---
allowed-tools: Bash(gh pr-review:*), Bash(gh repo view:*), Read, Edit, Write, Agent
description: Mine human PR review comments and append generalizable rules to the best-practices file
disable-model-invocation: false
---

You are mining human PR review comments from a single pull request and appending generalizable rules to the project's `code-review` best-practices artifact at `.claude/code-review/best-practices.md`.

The arguments passed to this command are: `$ARGUMENTS`.

Walk the steps below in order. Do not skip steps. Do not invent behaviors that are not here.

You have access to the bundled `gh-pr-review` skill (loaded with this plugin). Use it for every PR-review CLI invocation; do not re-document its flags here. Every `gh pr-review` invocation must follow the schema documented in that skill — review IDs are `PRR_…`, thread IDs are `PRRT_…`, comment node IDs are `PRRC_…`.

## Parse args

Strip leading/trailing whitespace from `$ARGUMENTS`. If the result is empty, print EXACTLY:

```
Usage: /gather-insight-pr <pr-url-or-number>
```

Then exit immediately. Do not proceed to any later step.

Otherwise treat the argument as either a PR URL (e.g. `https://github.com/<owner>/<repo>/pull/<n>`) or a bare PR number. Carry the argument into the next step as `<pr-arg>`.

## Pre-check

Run Bash:

```bash
test -f .claude/code-review/best-practices.md && echo OK || echo MISSING
```

If the output is `MISSING`, print EXACTLY:

```
No best-practices file found at .claude/code-review/best-practices.md.
Run /init first to create it.
```

Then exit immediately. Do not proceed to any later step.

If the output is `OK`, read the artifact's frontmatter:

```bash
awk '/^---$/{i++; next} i==1' .claude/code-review/best-practices.md
```

Parse the YAML mentally. Extract:

- The full section list — the union of the seven defaults (`security`, `bugs`, `performance`, `simplicity`, `testing`, `error_handling`, `conventions`) plus any additional slugs under the frontmatter `sections:` map. The frontmatter is the source of truth for the section list — do not invent slugs that aren't there.
- `skip_review_authors`: a list of reviewer logins to drop in addition to the bot list. If the key is absent, default to an empty list.

## Resolve repo

If `<pr-arg>` is a URL, parse `<owner>/<repo>` directly from the URL path (e.g. `https://github.com/foo/bar/pull/42` → `foo/bar`). The PR number is the last path segment.

If `<pr-arg>` is a bare number, resolve the current repo with:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

The result has the shape `<owner>/<repo>`. The PR number is `<pr-arg>` itself.

Carry `<owner>/<repo>` and `<pr-number>` into the next step.

## Fetch reviews

Use the bundled `gh-pr-review` skill — single call:

```bash
gh pr-review review view -R <owner>/<repo> --pr <pr-number>
```

The output is structured JSON with reviews, top-level review summary bodies, inline thread comments, and reply chains. Capture the JSON. Parse with `jq` or pipe directly into your reasoning.

If the call fails (PR not found, repo not accessible, etc.), surface the error verbatim to the user and exit. Do not fabricate results.

## Iterate

Iterate over `reviews[].comments[]`. This includes both:

- Top-level review summary bodies (the review's own `body` if present, with no associated `path`/`line` — treat these as "review summary" comments).
- Inline thread comments and reply chains (each with `path`, `line`, `author_login`, `body`).

Build a single working list of candidate comments. For each candidate, capture: `author_login`, `body`, `path` (or `none` for review summaries), `line` (or `0` for review summaries), and a flag indicating whether it is an inline thread comment or a review summary.

## Filter authors

For each candidate, drop the comment if **any** of the following match its `author_login` (case-insensitive):

- The login ends in `[bot]`. This is the future-proof rule — GitHub names every GitHub App with a `[bot]` suffix, so this catches all of them (including new bot variants we haven't enumerated below).
- The login matches one of the exact-known bot logins below via **full-string equality** (case-insensitive). Substring matching is forbidden here — `dependabot-watcher`, `mr-codeguru`, and `coderabbit-fan` are real human handles that must survive this filter. The exact-known bot logins are:
  - `coderabbitai`
  - `sourcery-ai`
  - `codiumai`
  - `codeball`
  - `codeguru-reviewer`
  - `github-actions[bot]`
  - `dependabot[bot]`
  - `renovate[bot]`
- The login appears in the project's `skip_review_authors` list (parsed from frontmatter; default empty). Match this list with full-string equality (case-insensitive) — same rule as above; no substring matching.

**Do not drop self.** The user often reviews other people's PRs and those comments are first-class rule sources. Self-authored comments must be preserved through this filter — never exclude the self login from the candidate set.

Track the count of distinct human reviewers (`author_login` values) that survive this filter — you will need it for the batch summary.

## Filter substantive

For each remaining candidate, drop the comment if **any** of the following match its `body`:

- `len(body.strip()) < 20`.
- The body contains no alphabetic character at all (emoji-only or punctuation-only).
- The body's substantive content is just an approval or a nit. Match case-insensitively, with **word boundaries**, against any of: `lgtm`, `looks good`, `looks good to me`, `approved`, `approve`, `nit`, `nit:`, `+1`, `:+1:`. Word boundaries mean: `nit:` matches and is dropped; `nittpick` does NOT match `nit` (substring would, but word boundary doesn't); `unit-test` does NOT trigger via `nit`.

For the approval/nit substrings, use judgment to avoid over-filtering: if a longer comment merely contains `looks good` followed by substantive reasoning (e.g. `this looks good and the implementation is clear because <reasoning>`), do **not** drop it. The simplest workable rule: drop only when the matched substring (with word boundaries) constitutes the *entire* substantive content of the body after stripping punctuation and whitespace. If the comment is materially longer than the matched approval phrase and carries reasoning, keep it for extraction.

The rule extractor itself decides if the comment yields a generalizable rule via the `generalizability` score in the next step — the substantive filter is just a cheap pre-filter, not the gate.

## Extract rules

For every comment that survives both filters, dispatch an `Agent` call to extract a structured rule. **Issue all `Agent` tool calls in a single response message** so they run concurrently — do not chain them sequentially.

For each `Agent` call:

- `subagent_type`: `general-purpose`
- `model`: `sonnet` (extraction needs reasoning but doesn't need opus)
- `description`: `Extract rule from PR comment`
- `prompt`: include all of the following:
  - The full comment `body`.
  - The comment's `path`, `line`, `author_login`, and PR number `<pr-number>` (for the evidence string).
  - A flag indicating whether the comment is an inline thread comment or a review summary. If it is a review summary, instruct the agent to use `path: none` and `line: 0` in the evidence and to note in the evidence that it is a review summary, not an inline thread.
  - The full list of artifact section slugs from the frontmatter (the seven defaults plus any custom sections). The agent must pick its `suggested_section` from this list — no inventing slugs.
  - The instruction to output exactly this YAML schema:

    ```yaml
    rule: <one-line imperative sentence>
    why: <one sentence rationale>
    evidence: "Reviewer @<login> on <path>:<line> in PR #<n>"
    suggested_section: <slug from the available list>
    generalizability: <0-100 integer>
    ```

  - The generalizability rubric:
    - 0-20: hyper-specific (e.g. "rename `foo` to `bar` here", "fix this typo on line 12").
    - 30-50: somewhat applicable but bound to this codebase (project-specific naming or layout).
    - 60-79: applicable to similar codebases or patterns (a real engineering rule, but bounded).
    - 80-100: broadly applicable engineering principle (would hold across many projects).
  - The instruction: if the comment is unclear, off-topic, or doesn't yield a generalizable rule, return `rule: null` (and any other fields are fine). The orchestrator will discard such results — never invent a rule from an unclear comment just to fill the slot.
  - The instruction: never write a `rule` without a `why`. Both must be derived from the comment's actual content.

Collect all returned extractions into a single list once the parallel batch completes.

## Filter generalizability

Drop every extraction where any of the following hold:

- `rule: null` (the agent could not derive a generalizable rule).
- `generalizability < 60`.

The threshold is fixed at 60. Never lower it without explicit user instruction.

## Dedup

For each surviving extraction, compare it against the existing rules in the artifact's body for its `suggested_section`. To find the section's body region, follow the same anchor logic as `/gather-insight-discussion`:

1. Read the artifact body (everything after the closing `---` of the frontmatter).
2. Enumerate every `## ` heading in the body.
3. Slugify each heading: lowercase, whitespace runs → `_`, drop any character that is not `[a-z0-9_]`.
4. Pick the heading whose slugified form equals the extraction's `suggested_section`. The body region runs from the line after that heading to the line before the next `## ` heading or EOF.
5. Identify existing rule bullets in the region — top-level lines starting with `- **`. Ignore the placeholder comment (`<!-- Add inline rules here, or rely on refs above -->`) and any sub-bullets (`  - **Why:**`, `  - **Example:**`).

Apply the same similarity anchor as `/gather-insight-discussion`: the existing rule covers the same imperative subject as the new rule when they share the primary verb plus the primary noun phrase. Examples: "always validate input at trust boundaries" and "check user input at API entry points" are matches (verb: validate/check, noun phrase: input at boundaries). "always check for null" and "always validate input" are not matches (different noun phrases). Wording differences alone don't count; the underlying subject must overlap.

Bucket each extraction:

- `new` — no overlap with any existing rule in the section.
- `duplicate` — substantively identical to an existing rule. Drop silently; do not surface in the per-rule walkthrough.
- `refines-existing` — overlaps with an existing rule but adds nuance (e.g. the new rule narrows the condition, names a specific failure mode, or adds rationale the existing rule lacks). Surface to the user and offer to replace.

## Batch summary

Print EXACTLY:

```
Mined N comments from M reviewers (excluding bots and skip_review_authors).
Found: X new, Y refining, Z duplicates (skipped).
```

Where:
- `N` is the count of comments that survived **both** the author and substantive filters (i.e. comments dispatched to extraction agents).
- `M` is the distinct count of `author_login` values among those `N` comments.
- `X`, `Y`, `Z` are the bucket counts after generalizability filtering and dedup.

Then prompt EXACTLY:

```
Review the X new + Y refining rules? (a)ccept all / (s)elect / (n)one [s]
```

Substitute the actual `X` and `Y` integers in the prompt. Parse the reply (case-insensitive single character; empty defaults to `s`):

- `a` (accept all): apply every `new` and `refines-existing` rule without per-rule confirmation.
- `s` (select, default): walk the rules one at a time per the per-rule walkthrough below.
- `n` (none): skip everything; jump straight to the final summary with `Added 0 rules`.

If `X + Y == 0` and the user has nothing to review, skip the prompt entirely and jump straight to the final summary.

## Apply

### Per-rule walkthrough (default `s`)

For each rule in the combined `new` + `refines-existing` list, in the order they were extracted:

1. Print the rule, its `why`, its `evidence`, the `suggested_section`, and the `bucket`. For `refines-existing`, also print the closest matching existing rule (with its line number) so the user knows what they're refining.

   Example shape:

   ```
   Rule: <imperative sentence>
   Why: <one-sentence rationale>
   Evidence: <evidence string>
   Suggested section: <slug>
   Bucket: <new|refines-existing>
   Refines: L<n>: <existing rule text> ...   (only when bucket is refines-existing)
   ```

2. Prompt EXACTLY:

   ```
   (y)es / (e)dit / (n)o
   ```

   Parse the reply (case-insensitive single character; empty defaults to `n`):

   - `y` (yes): apply the rule per "Apply semantics" below.
   - `e` (edit): enter inline-edit mode. Ask which field the user wants to revise:

     ```
     Edit which? (r)ule / (w)hy / (a)ll
     ```

     The auto-generated `evidence` string (`Reviewer @<login> on <path>:<line> in PR #<n>`) is **not** editable here — it is the audit trail provenance and rewriting it would falsify the source attribution and break parsing. If the user wants to add their own context (e.g. a snippet of the reviewer's comment), they can edit the artifact directly afterward.

     Re-prompt for the chosen field(s), update the in-memory state, then re-render the rule block and re-ask `(y)es / (e)dit / (n)o`. Loop until the user picks `y` or `n`.
   - `n` (no): skip this rule. Do not write anything.

### Accept all (`a`)

Apply every rule in the `new` + `refines-existing` list without per-rule confirmation, in the order they were extracted, using "Apply semantics" below.

### None (`n`)

Skip everything. Jump straight to the final summary with `Added 0 rules`.

### Apply semantics

For each rule the user accepts (whether via `accept all` or per-rule `y`), use `Edit` to apply the change to `.claude/code-review/best-practices.md`. Anchor the section's body region the same way as in the dedup step — scan the body for `## ` headings, slugify each, and use the heading whose slugified form equals the rule's `suggested_section` as the anchor. Do not reconstruct the title from the slug.

The semantics depend on the bucket:

- **`new`** bucket — append the new rule as a bullet at the end of the section's body region. Apply the placeholder rule:
  - **Placeholder rule:** if the section currently contains only the placeholder comment `<!-- Add inline rules here, or rely on refs above -->` (and nothing else), **replace** the placeholder line with the new bullet block. The placeholder must be removed once a section has at least one rule.
  - **Otherwise:** keep the existing rules in place and append the new bullet block after the last existing rule, immediately before the next `## ` heading (or before EOF if the target section is the last section).

- **`refines-existing`** bucket — locate the matched existing bullet (and its `**Why:**`/`**Example:**` sub-bullets, which are the indented lines immediately following) and replace that whole bullet block with the new bullet block. Never silently rewrite an existing rule — the per-rule confirmation in the walkthrough above is what authorizes this; on `accept all`, the user has already given blanket consent. If multiple existing rules were tagged as overlapping, replace only the closest match.

Use the canonical rule format:

```markdown
- **<rule sentence>**
  - **Why:** <reason>
  - **Example:** <evidence>
```

The `**Example:**` sub-bullet is built from the rule's `evidence` field. Format it as:

```
**Example:** Reviewer @<login> on <path>:<line> in PR #<n> — "<short snippet of the comment if it adds context>"
```

Optionally include a short snippet (one sentence at most) of the original comment body if it materially clarifies the rule. If the snippet would just restate the rule, omit it and use only the evidence string. For comments that are entire review-summary bodies (top-level rather than inline), use `path:none, line:0` in the evidence and note it is a review summary, not an inline thread:

```
**Example:** Reviewer @<login> review summary on PR #<n>: "<short snippet>"
```

**Preserve placeholder comments in all sections you are not modifying.** Only the section a new rule lands in loses its placeholder (and only if the section had nothing else). Never strip placeholders from sections that received no new rules in this run.

## Summary

After all writes complete (or after `none` was selected), print EXACTLY:

```
Added K rules across <comma-separated list of slugs> from PR #<n>.
```

Where:
- `K` is the count of rules actually written to the artifact (not the count surfaced in the batch summary — only the ones the user accepted, including refinements that replaced existing bullets).
- `<comma-separated list of slugs>` is the deduplicated set of `suggested_section` slugs the rules landed in, in the order they were first written. If `K == 0`, render the list as `none` (e.g. `Added 0 rules across none from PR #42.`).
- `<n>` is the PR number resolved earlier.

## Hard rules

These constraints are non-negotiable. Re-read them before each step:

- **Drop bots and configured `skip_review_authors`. Do NOT drop self.** Self-authored review comments are first-class rule sources and must always survive the author filter.
- **Never lower `generalizability < 60` threshold without explicit user instruction.** The threshold is fixed.
- **Never invent a rule from an unclear comment.** If extraction returns `rule: null`, discard the result. Do not synthesize a rule just to fill the slot.
- **Never write a rule without a Why.** Both `rule` and `why` must be derived from the comment's actual content. Drop the extraction if either is missing.
- **Never silently rewrite existing rules.** `refines-existing` requires user confirmation in the per-rule walkthrough, or blanket `accept all` consent. The dedup step alone never edits the artifact.
- **Preserve the placeholder comment** (`<!-- Add inline rules here, or rely on refs above -->`) in all sections you are not modifying. Only the section a rule lands in loses its placeholder, and only if it had nothing else.
- **Use the bundled `gh-pr-review` skill.** Every `gh pr-review` invocation must follow the schema documented in the skill — review IDs `PRR_…`, thread IDs `PRRT_…`. Do not re-document its flags inline; lean on the skill.
- **Use only the declared tools:** `Bash` (scoped to `gh pr-review:*`, `gh repo view:*`), `Read`, `Edit`, `Write`, `Agent`. Do not request anything else.
