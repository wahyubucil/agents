---
allowed-tools: Bash(gh pr-review:*), Bash(gh pr:*), Bash(gh api:*), Bash(gh repo:*), Bash(git:*), Read, Glob, Grep, Agent
description: Run a project-aware code review and post a pending GitHub review with inline comments
disable-model-invocation: false
---

You are running a project-aware code review on a single pull request. The output is a **pending** GitHub review with inline comments and a draft summary body — you never auto-submit. The user reviews the pending review and decides whether to submit.

The arguments passed to this command are: `$ARGUMENTS`.

You have access to the bundled `gh-pr-review` skill (loaded with this plugin). Use it for every PR-review CLI invocation; do not re-document its flags here. Every `gh pr-review` invocation must follow the schema documented in that skill — review IDs are `PRR_…`, thread IDs are `PRRT_…`, comment node IDs are `PRRC_…`.

Walk the 14 steps below in order. Do not skip steps. Do not invent behaviors that are not here. The "Hard rules" block at the bottom is non-negotiable — re-read it before each step.

## Resolve PR

Strip leading/trailing whitespace from `$ARGUMENTS`. Parse the result:

- **URL form** (e.g. `https://github.com/<owner>/<repo>/pull/<n>`): parse `<owner>/<repo>` directly from the URL path; the PR number is the last path segment.
- **Bare integer** (e.g. `42`): the PR number is the argument itself; resolve the current repo with:

  ```bash
  gh repo view --json nameWithOwner -q .nameWithOwner
  ```

  The result has the shape `<owner>/<repo>`.
- **Empty**: auto-detect from the current branch:

  ```bash
  gh pr view --json number,headRepository -q .number
  ```

  Resolve the repo with `gh repo view --json nameWithOwner -q .nameWithOwner` (or read it from the same `gh pr view` call by extending the `--json` argument).

If neither an explicit argument nor a current-branch PR resolves, print EXACTLY:

```
Usage: /review [<pr-url-or-number>]
No PR found for the current branch and no PR argument provided.
```

Then exit immediately. Do not proceed to any later step.

Carry `<owner>/<repo>` and `<pr-number>` into the next step. Do not refetch them — every later step references the same values.

## Pre-flight

Dispatch a **single** `Agent` tool call (a Haiku sub-agent) to check eligibility. Use:

- `subagent_type: general-purpose`
- `model: haiku`
- `description:` `Pre-flight eligibility check`
- `prompt:` includes:
  - The PR number `<pr-number>` and repo `<owner>/<repo>`.
  - The artifact's `skip_authors` list (if you have already loaded the artifact you can pass it; otherwise pass the default `[dependabot[bot], renovate[bot], github-actions[bot]]` — step 4 will load the real list later, but pre-flight is first by design so it must run with whatever skip list you have, falling back to the bot defaults if the artifact has not been read yet).
  - Instruction: run the following Bash and analyze its output:

    ```bash
    gh pr view <pr-number> -R <owner>/<repo> --json state,isDraft,author,reviews,headRefOid
    ```

  - Decision rules (apply in order; first match wins):
    - If `state` is `CLOSED` or `MERGED`: decision = `SKIP`, reason = `PR is <state>`.
    - If `isDraft` is `true`: decision = `SKIP`, reason = `PR is a draft`.
    - If `author.login` is in the passed-in `skip_authors` list (case-insensitive, full-string equality): decision = `SKIP`, reason = `Author @<login> is in skip_authors`.
    - Otherwise: decision = `PROCEED`. If `reviews[]` contains any review whose author is the current user (resolve self via `gh api user --jq .login` if needed), record the most recent such review's `commit.oid` (or equivalent `headRefOid`-anchor field) as `prior_review_commit_sha`. If no prior self-review, `prior_review_commit_sha = null`.
    - Always record the PR's current `headRefOid` as `head_sha`.
  - Output schema (return as a single JSON object):

    ```yaml
    decision: SKIP | PROCEED
    reason: <text>
    prior_review_commit_sha: <sha-or-null>
    head_sha: <sha>
    ```

On the sub-agent's return:

- If `decision == SKIP`: print the reason verbatim (e.g. `Skipping: PR is MERGED`) and exit. Do not proceed past this step.
- If `decision == PROCEED`: continue to Delta computation. Carry `prior_review_commit_sha` and `head_sha` into later steps.

Do not re-implement these checks inline; they belong to the sub-agent so they run on Haiku, not in the orchestrator.

## Delta

If `prior_review_commit_sha` is `null`, the **full PR diff is in-scope**. Skip the rest of this step.

If `prior_review_commit_sha` is set, compute the delta:

1. Best-effort fetch (silent on failure):

   ```bash
   git fetch --quiet 2>/dev/null || true
   ```

2. Compute the delta diff scoped to the changed files:

   ```bash
   git diff <prior_review_commit_sha>..<head_sha>
   ```

   You may filter by changed files with `git diff <prior>..<head> -- <files>` if you want to narrow further; this is the **delta diff** and is the in-scope diff for the rest of the command.

3. If the delta diff is empty (the PR head has not advanced since the prior review), print EXACTLY:

   ```
   already reviewed at HEAD
   ```

   Then exit. Do not proceed to any later step.

The delta diff (or the full PR diff when no prior review exists) is the **in-scope diff** that all section agents will see, and the only diff against which inline comment anchors may be placed.

## Load artifact

Read `.claude/code-review/best-practices.md`. Use the following Bash to test for existence first:

```bash
test -f .claude/code-review/best-practices.md && echo OK || echo MISSING
```

If the file is `MISSING`, print EXACTLY:

```
Warning: no best-practices.md found at .claude/code-review/. Running in degraded mode (false-positive baseline + confidence rubric only, no project rules).
```

Then continue with an empty sections dict. The review will spawn a **single default Sonnet section agent** in step 7 with a stripped-down prompt that just looks for obvious bugs (no project-specific rules). Use `min_confidence: 80` as the fallback threshold and `[dependabot[bot], renovate[bot], github-actions[bot]]` as fallback `skip_authors`. Do not abort the review.

If the file exists, parse its YAML frontmatter (the block between the first two `---` lines):

```bash
awk '/^---$/{i++; next} i==1' .claude/code-review/best-practices.md
```

Extract:

- `sections:` — a map from slug to `{refs: [...], model: <inherit|sonnet|opus|haiku>}`.
- `min_confidence:` — integer (default 80 if absent). If `min_confidence` is missing OR is not a non-negative integer in `[0, 100]` (e.g. a typo like `high`, or a string like `"80"`), warn `min_confidence is missing or invalid; defaulting to 80` and use 80.
- `skip_authors:` — list of logins (default to the bot list if absent).

The seven default slugs (`security`, `bugs`, `performance`, `simplicity`, `testing`, `error_handling`, `conventions`) plus any custom slugs in the map form the full section list.

## Resolve refs

For each section in the frontmatter `sections:` map:

1. **Read the section body** from the artifact body. Anchor by slug:
   - Read the body (everything after the closing `---` of the frontmatter).
   - Enumerate every line that starts with `## ` (exactly two hashes and a space).
   - Slugify each heading: lowercase, whitespace runs → `_`, drop any character that is not `[a-z0-9_]`.
   - Pick the heading whose slugified form equals the section slug. The body region runs from the line after that heading to the line before the next `## ` heading or EOF.
   - If no body heading slugifies to the section slug (artifact inconsistent), warn (`section <slug>: no body heading found, using empty body`) and treat the body as empty for this section. Do not bail the whole review on one malformed section.

2. **Resolve each `ref` in the section's `refs` list:**
   - If the ref starts with `skill:<name>` (e.g. `skill:owasp-defenses`), resolve in **this exact order**, taking the first hit:
     1. `.claude/skills/<name>/SKILL.md`
     2. `.agents/skills/<name>/SKILL.md`
     3. `~/.claude/skills/<name>/SKILL.md`
     4. `~/.claude/plugins/cache/*/*/skills/<name>/SKILL.md` (glob; first match)
   - Otherwise treat the ref as a file path **relative to the project root**. Read it directly.
   - On a miss for any ref (no path resolved, or file unreadable), print EXACTLY: `ref not resolved: <ref>` and continue with empty content for that ref. Do not abort.

3. **Skip the section entirely** (no agent dispatch in step 7) if **both** of the following hold:
   - The section's body is **only** the placeholder comment `<!-- Add inline rules here, or rely on refs above -->` (modulo whitespace).
   - The section's `refs` list is empty.

   This is the "inactive section" rule — empty sections waste an agent slot.

4. Bundle each surviving section as `{section_slug, model, body, refs_content, ref_paths}`. `refs_content` is the concatenation of resolved ref contents (with each prefixed by a small header naming the source path, so the agent can cite which ref grounded a finding); `ref_paths` is the list of resolved paths for traceability.

## Shared context

Fetch once, share to all section agents in step 7. Scope every fetch to the in-scope diff (the delta diff if step 3 produced one, else the full PR diff).

1. **Diff:**
   ```bash
   gh pr diff <pr-number> -R <owner>/<repo>
   ```
   When delta-scoped, use the precomputed `git diff <prior>..<head>` (optionally narrowed by changed files: `git diff <prior>..<head> -- <files>`) instead. Either way you produce a single diff string that is the in-scope diff for the rest of the run.

2. **Git blame for each changed hunk.** For every modified hunk in the in-scope diff, run:
   ```bash
   git blame -L <hunk_start>,<hunk_end> -- <path>
   ```
   Truncate the blame output to the **last 5 unique commits per hunk** (concise authorship signal — full blame is too noisy).

3. **In-code comments in modified files.** Restrict to files that appear in the in-scope diff:
   ```bash
   grep -n -E '(TODO|FIXME|NOTE|HACK|@see|@deprecated)' <changed-files>
   ```
   Capture the matching lines verbatim. These are hints about intent that section agents may use to ground or reject findings.

4. **Past PR comments on touched files.** For each in-scope file path, prefer the `path:` qualifier first; fall back to a keyword (the file name without extension) only if zero hits.

   GitHub PR search supports `path:` to scope to PRs that touched a given file path. Prefer this over keyword search to avoid noisy matches on common filenames.

   ```bash
   # Try path-qualified search first:
   gh pr list --state merged --search "path:<file-path>" --limit 5 --json number,title,files

   # If empty, fall back to a keyword (filename without extension), but warn that results may be noisy:
   gh pr list --state merged --search "<keyword>" --limit 5 --json number,title,files
   ```
   For each of the top 5 matching merged PRs, fetch its review comments via:
   ```bash
   gh pr-review review view -R <owner>/<repo> --pr <past-pr-number> --not_outdated --tail 3
   ```
   We want the original review comment plus a small reply window for context — not just the latest reply. Concatenate inline comments. **Cap the total at ~50 comments** across all touched files to keep the agents' context manageable; drop the oldest if you exceed the cap.

Bundle everything into a single shared-context blob: `{diff, blame_per_hunk, in_code_comments, past_pr_comments}`. This blob is passed verbatim to every section agent in step 7.

## Section agents

**Issue all `Agent` tool calls in a single response message** so they run concurrently (per the Agent tool's parallel-dispatch semantics). Do not chain them sequentially.

If step 4 ran in degraded mode (no artifact), spawn **one default agent** instead of per-section agents. Its prompt omits the section heading/body/refs/ref-paths placeholders; the section slug is `default`, the model is `sonnet`, the body rule is `No project-specific rules — focus on obvious bugs introduced by the diff`. The false-positive baseline, confidence rubric, build/typecheck/lint disclaimer, and output schema below still apply.

For each section bundle from step 5 (or the single default bundle in degraded mode), dispatch one `Agent` call with:

- `subagent_type: general-purpose`
- `description:` `<section_slug> review`
- `model:` —
  - if the bundle's `model` is `sonnet`, `opus`, or `haiku`, pass that string verbatim;
  - if the bundle's `model` is `inherit`, **omit the `model` parameter entirely** (the Agent tool inherits the parent model when the parameter is absent — `inherit` is **not** a literal value the tool accepts).
- `prompt:` must include all of the following, in this order:

  1. **Section heading and body rules** — the user's project rules for this section. Format:

     ```
     # Section: <Title>

     <body verbatim>
     ```

  2. **Reference materials** — the resolved refs content from step 5, each clearly labeled with its source path. Format:

     ```
     # Reference materials

     ## <ref-path-1>
     <content>

     ## <ref-path-2>
     <content>
     ```

  3. **The in-scope diff** (delta-scoped when applicable):

     ```
     # Diff
     <diff verbatim>
     ```

  4. **Shared context** — blame, in-code comments, past PR comments:

     ```
     # Shared context

     ## Git blame per hunk
     <blame summary, ≤5 unique commits per hunk>

     ## In-code comments in modified files
     <grep output>

     ## Past PR comments on touched files
     <up to 50 inline comments concatenated>
     ```

  5. **The false-positive baseline (verbatim, no edits):**

     ```
     Drop findings that are:
     - Pre-existing issues not introduced by the PR
     - Looks like a bug but is not
     - Pedantic nitpicks a senior engineer would not raise
     - Linter / typechecker / compiler territory (CI handles it)
     - General quality issues (test coverage, doc completeness) unless a rule explicitly demands them
     - Issues silenced via lint-ignore / eslint-disable comments
     - Likely intentional changes
     - Issues on lines the user did not modify
     ```

  6. **The confidence rubric (verbatim):**

     ```
     0:   false positive, fails light scrutiny, or is pre-existing
     25:  might be real, agent could not verify
     50:  real but minor or rare in practice
     75:  real, important, very likely to bite, or directly cited by a rule
     100: real, will happen frequently, evidence directly confirms
     ```

  7. **Build/typecheck/lint disclaimer (verbatim):**

     ```
     Do not run build, typecheck, or lint. CI handles those.
     ```

  8. **Output schema instruction:**

     ```yaml
     findings:
       - path: <relative path>
         line_start: <int>
         line_end: <int>
         message: <imperative one-liner — what to do>
         evidence: <optional body of the comment — see style rules below; omit/empty when not needed>
         confidence: <0-100>
     ```

     Followed by: `If no findings, return findings: [].`

  9. **Comment style rules (verbatim):**

     ```
     - `message` is imperative and short. State what to do.

     - `evidence` is OPTIONAL. Leave it empty when `message` plus the anchored code
       already makes the change obvious. Most simple fixes (rename, flip a condition,
       remove an unused arg, reorder two lines) need no body. Only add a body when
       the WHY isn't obvious from message + code alone.

     - When you do write a body, keep it conversational and simple (1–3 short
       sentences). Talk about concrete impact or the convention plainly. Do NOT
       cite specific files or sections — no "AGENTS.md §2 mandates...", no
       "matches `path:Lx-Ly` precedent", no "violates the <section-slug> rule X".
       Write the way a senior reviewer would in a quick comment, not the way a
       static analyzer surfaces a rule ID. For complex fixes, include a fenced
       code block showing the concrete change.

     - Do not repeat the message in the body. Do not list multiple separate angles.
       Do not append a separate "Evidence:" paragraph — citations live inline in the
       prose, not as a footer.

     - When the fix is a literal in-place replacement of the anchored lines, prefer
       GitHub's suggestion block:

           ```suggestion
           <full replacement for the anchored line range, indentation included>
           ```

       Use `suggestion` only when ALL of these hold:
         1. The block replaces the EXACT anchored line range (line_start..line_end),
            no more and no less.
         2. The replacement indentation matches the existing file.
         3. The replacement is a complete, committable unit — not a sketch.
       If any condition fails, use a plain ```<lang> block instead. A plain block
       illustrates; a suggestion block is a one-click commit. Don't conflate them.

     - One suggestion block per comment. GitHub only applies the first.
     ```

Once all parallel `Agent` calls complete, collect every finding into a flat list. **Tag each finding with the originating section slug** (the merge step uses this to build the merged finding's `contributing_sections` list, which the posting template renders in the comment footer). Findings outside that schema are ignored.

## Merge

Walk the flat finding list and group findings whose `(path, line_start, line_end)` overlap within ±2 lines on either edge. For each group:

- Keep the **highest-confidence** finding as the canonical one.
- **Build a `contributing_sections` list** on the canonical finding: an ordered, deduplicated list starting with the canonical finding's section slug, followed by each other contributor's section slug in confidence-descending order. If two contributors share the same confidence, preserve their relative order from the pre-merge flat list. The posting template renders this list in the comment footer (e.g. `bugs · security`).
- **Concatenate non-empty evidence strings with `\n\n`** (a blank line between paragraphs), in confidence-descending order. Skip contributors whose `evidence` is empty/null — they contributed only their section slug to `contributing_sections`. If every contributor's `evidence` is empty, the merged finding's `evidence` is empty too. Inline ` | ` joins read awkwardly when evidence contains multi-sentence prose or fenced code blocks; double-newline joins read like consecutive paragraphs.
- **One code block per merged comment.** GitHub renders only the first `suggestion` block, and stacking plain code blocks from different sections is noisy. Rule: if two or more contributing findings each include a fenced code block (any language, including `suggestion`), keep the canonical finding's evidence verbatim; from non-canonical contributors, **strip every fenced code block before concatenation** (delete the entire ` ```...``` ` span) but keep the surrounding prose. The canonical entry's code block survives intact. If 0 or exactly 1 contributor has a code block, no stripping is needed — the surviving block (canonical or not) passes through.

This collapses cross-section duplicates (e.g. `bugs` and `security` both flagging a missing null check on the same line). Do not lose prose evidence — every contributor's reasoning still reaches the merged comment.

## Filter

Drop every merged finding whose `confidence` is **strictly less than** `min_confidence` from the artifact frontmatter (or 80 in degraded mode).

**Never lower the threshold below the artifact's `min_confidence` value.** This is a hard rule; the user's threshold is authoritative.

If 0 findings remain after filtering: still proceed with steps 11-14. Open the pending review, add no inline comments, and write a summary body that states `Reviewed sections: ...; no high-confidence findings.` This gives the user a posting trace and lets them submit/discard the empty review on GitHub. The Submit-now? prompt remains.

## Post-eligibility

Before posting anything, dispatch a **second** `Agent` call with the same logic as the Pre-flight step (model `haiku`, same Bash, same skip-author list). If the PR's state has changed mid-review (closed, merged, or converted to draft) **abort posting** and print:

```
PR state changed during review (<reason>). Aborting post; no pending review created.
```

This is the race guard — without it a long-running review may post comments to a PR that no longer accepts them. If the second check returns `PROCEED`, continue.

## Pending review

**Pre-step 11: Verify `gh pr-review` extension is installed.**

```bash
gh extension list 2>/dev/null | grep -q pr-review || { echo 'Missing dependency: gh extension install agynio/gh-pr-review'; exit 1; }
```

If missing, print the install command and exit. The earlier `gh pr-review review view` calls (steps 6, 10) would have already failed if it were missing — this is a defense-in-depth check before the harder-to-reverse `--start` operation.

Open the pending review:

```bash
gh pr-review review --start \
  --pr <pr-number> -R <owner>/<repo> \
  --commit <head_sha>
```

Capture the returned review ID. It will look like `PRR_…` — that is the literal prefix the `gh-pr-review` skill documents. Carry it as `<review-id>` for the rest of the run.

If the start command fails (network error, permissions), surface the error verbatim to the user and exit. Do not retry blindly; the user can re-run `/review`.

## Add inline comments

Note: the bundled `gh-pr-review` skill snapshot (v1.6.2 era) doesn't document `--start-line`, but the actual CLI supports it for multi-line ranges. Run `gh pr-review review --add-comment --help` to confirm the flag is available on the installed version.

For each surviving finding, add an inline comment to the pending review:

```bash
gh pr-review review --add-comment \
  --review-id <review-id> \
  --pr <pr-number> -R <owner>/<repo> \
  --path <path> \
  --line <line_end> \
  --start-line <line_start> \
  --body "$(cat <<EOF
<message>

<evidence>

---
🤖 _Agent review — <sections>_
EOF
)"
```

**Body format (must follow exactly):**

- `<message>` line: the finding's `message` field — an imperative one-liner.
- Blank line.
- `<evidence>` paragraph(s): the merged evidence string from step 8 (paragraphs joined by blank lines; may include a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise). **If `evidence` is empty/null, omit this paragraph and the blank line above it entirely** — the body collapses to `<message>`, the separator, and the footer.
- Blank line, then the literal separator `---` on its own line.
- Footer line: `🤖 _Agent review — <sections>_` where `<sections>` is the merged finding's `contributing_sections` list joined with ` · ` (space–middle-dot–space). Slugs render verbatim — lowercase, underscores intact, no title-casing. For a single-section finding this is just the one slug (e.g. `🤖 _Agent review — bugs_`).

**Anchoring rules:**

- Anchor lines must be **inside the in-scope (delta) diff**. Never anchor an inline comment to a line outside the delta diff — even if the finding cites an older line for context, the anchor must be on a delta line.
- If a finding needs to reference an older (pre-delta) line for context, mention the older line in the body text but anchor the comment on a delta line.
- If `line_end == line_start`, **omit `--start-line`** (single-line comments don't take a range).

If a single comment fails to post (e.g. the path moved outside the delta), log the failure (`failed to anchor: <path>:<start>-<end>`) and continue with the next finding. Do not abort the whole run on a single anchor failure.

## Summary

Compose a Markdown summary body. Do not post it yet — it is the draft the user can edit before submitting. Use this exact shape:

```markdown
### Automated review (Claude)

Reviewed sections: <comma-separated section slugs that ran>.
Findings: <N> at confidence ≥ <threshold> (from <total_candidates> raw).

Highlights:
- <link to top finding using full-SHA permalink format with ±1 line context>
- <link to second finding…>

🤖 Generated by code-review plugin (delta vs <prior_commit_short>) — based on `.claude/code-review/best-practices.md`.
```

Where:

- `<N>` is the count of inline comments actually posted in step 12.
- `<threshold>` is the `min_confidence` value used.
- `<total_candidates>` is the count of raw findings before merge+filter (step 7's flat list size).
- The "Highlights" bullets are the top 3 findings by confidence; render each as a Markdown link using the **full-SHA permalink format** with ±1 line of context:

  ```
  https://github.com/<owner>/<repo>/blob/<full-head-sha>/<path>#L<start>-L<end>
  ```

  Use the full HEAD SHA, not the short one. Pad `start`/`end` outward by 1 line each so the linked range shows ±1 line of context, but keep the comment anchor on the original lines.
- `<prior_commit_short>` is the first 7 characters of `prior_review_commit_sha`. If there is no prior review (full review path), the parenthetical reads `(full review)` instead of `(delta vs <prior_commit_short>)`.

For degraded mode, "Reviewed sections" is `default (no artifact)`.

This summary is **draft text** — store it; print it as part of step 14's prompt.

## Submit now

**Stop. Do not submit.** Print EXACTLY (substituting the captured values):

```
Pending review created with <N> inline comments.
GitHub URL: <pr-url>
Submit when ready:
  gh pr-review review --submit --review-id <review-id> --pr <pr-number> -R <owner>/<repo> \
    --event COMMENT|REQUEST_CHANGES|APPROVE \
    --body "<summary above, copy-paste from below>"

<summary body draft>

Submit? Reply with one of:
  n / no / cancel                          — don't submit (review stays pending on GitHub)
  approve / lgtm / ship it / looks good    — submit as APPROVE with the draft summary above as the body
  comment                                  — submit as COMMENT with the draft summary above as the body
  submit / send / post / request changes   — submit as REQUEST_CHANGES with the draft summary above as the body
```

Where `<pr-url>` is `https://github.com/<owner>/<repo>/pull/<pr-number>` and `<summary body draft>` is the full Markdown summary you composed in step 13 (rendered as-is, not in a fenced block — the user is going to copy-paste it into the `--body` argument).

Parse the user's reply against the **first non-empty line**, case-insensitive, trimmed. **First match wins** — apply the rules in this order:

1. **Cancel** — first line is empty or one of: `n`, `no`, `nope`, `cancel`. → exit cleanly. The pending review remains visible on GitHub for manual editing or submission later.
2. **APPROVE** — first line contains any of: `approve`, `lgtm`, `ship it`, `looks good`, `good to go`. → event = `APPROVE`.
3. **COMMENT** — first line is exactly one of: `comment`, `comments`, `comment only`. → event = `COMMENT`. (Exact-match, not `contains` — too easy for "submit a comment" or "approve, just one comment on line 4" to false-match otherwise.)
4. **REQUEST_CHANGES** — first line contains any of: `submit`, `send`, `post`, `request changes`, `request_changes`, `go ahead`. → event = `REQUEST_CHANGES`.
5. **Anything else** — echo what was received (truncate to ~60 chars if longer) and re-ask:

  ```
  Got "<received>", which I can't read as cancel/approve/comment/submit. Try again with one of:
    n          — don't submit
    submit     — submit as REQUEST_CHANGES (also "request changes", "send", "post")
    approve    — submit as APPROVE (also "lgtm", "ship it", "looks good")
    comment    — submit as COMMENT
  ```

The order matters: the approve check runs before request_changes, so "approve and submit" → APPROVE. The cancel match is exact (line is one of `n` / `no` / `nope` / `cancel` after trim), so a sloppy "no, send it" falls through to rule 4 and submits. If the user means cancel, they should type just `n` or `no`.

When a valid event is captured, run:

```bash
gh pr-review review --submit \
  --review-id <review-id> \
  --pr <pr-number> -R <owner>/<repo> \
  --event <chosen-event> \
  --body "<summary body draft>"
```

Surface the result. If submission fails, print the error and keep the pending review (the user can submit manually).

After this step the run is complete. Do not loop, do not re-spawn agents.

## Hard rules

These constraints are non-negotiable. Re-read them before each step:

- **Never auto-submit.** Always stop at "Submit?" (step 14). The pending review must be visible on GitHub before the user has a chance to approve submission. The only path that runs `gh pr-review review --submit` is when the user replies in step 14 — see step 14 for what counts as a valid reply. Loose phrasing is allowed; the reply must come from the user *post-prompt*, not be inferred from the original question. **Stop. Do not submit.**
- **Never lower the `min_confidence` threshold below the artifact's value.** The user's threshold (or 80 in degraded mode) is authoritative; do not relax it because findings are sparse.
- **Never anchor an inline comment to a line outside the delta diff.** If a finding cites an older pre-delta line for context, mention it in the body text but anchor the comment on a line inside the delta diff. Lines outside the in-scope diff are off-limits for anchors.
- **Never run build, typecheck, or lint.** CI handles those. The build/typecheck/lint disclaimer must appear verbatim in every section agent's prompt.
- **Never proceed past pre-flight if eligibility says SKIP.** The pre-flight sub-agent's `decision` is binding — print the reason and exit.
- **Re-check eligibility before posting.** The post-eligibility sub-agent (step 10) is the race guard against the PR being closed/merged/drafted mid-review. Without it, you can leak comments onto a PR that no longer accepts them.
- **Use the bundled `gh-pr-review` skill semantics.** Review IDs are `PRR_…`; thread IDs are `PRRT_…`. Do not re-document the CLI flags inline; lean on the skill.
- **Issue parallel section-agent dispatches in a single response message.** All `Agent` tool calls for step 7 must be batched together so they run concurrently. Sequential dispatch is wrong.
- **For `model: inherit` sections, omit the Agent tool's model parameter.** Do not pass `inherit` as a literal — the tool does not accept that string. Omit the model parameter entirely and the tool inherits from the parent.
- **Use only the declared tools:** `Bash` (scoped to `gh pr-review:*`, `gh pr:*`, `gh api:*`, `gh repo:*`, `git:*`), `Read`, `Glob`, `Grep`, `Agent`. Do not request anything else.
