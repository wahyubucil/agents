---
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
description: Distill a rule from a discussion and append it to the best-practices file
disable-model-invocation: false
---

You are distilling a single rule from a focused discussion and appending it to the project's `code-review` best-practices artifact at `.claude/code-review/best-practices.md`.

The arguments passed to this command are: `$ARGUMENTS`.

Walk the steps below in order. Do not skip steps. Do not invent behaviors that are not here.

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

If the output is `OK`, continue.

## Topic seed

If `$ARGUMENTS` is non-empty (after stripping leading/trailing whitespace), treat it as the seed topic for the dialogue and skip the topic prompt — the user has already told you what they want to formalize.

If `$ARGUMENTS` is empty, prompt the user EXACTLY:

```
What rule do you want to formalize?
```

Wait for a reply. Treat the reply as the seed topic.

## Dialogue

Run a focused dialogue. Ask only the questions necessary to extract the three required outputs below. Keep it tight — do not interrogate the user; clarify only when their answers are unclear or contradictory.

You must extract:

1. **Rule** (mandatory): one clear sentence in **imperative form**. Examples: `Always validate input at trust boundaries.` / `Never log secrets.` / `Prefer pure functions for business logic.` If the user's seed topic already reads as a usable imperative sentence, you may confirm it back to them rather than re-asking.
2. **Why** (mandatory): one sentence explaining the rationale — *why* this rule exists, not what it does. Ask the user explicitly. **Never invent a Why.** If the user cannot articulate a reason, do not write the rule — print:

   ```
   No Why provided. Skipping — a rule without a Why won't be written.
   ```

   Then jump to the loop step (skip the rest, ask `Another? (y/n) [n]`).

3. **Example** (optional, off by default): a short code or text snippet illustrating the rule. Ask once:

   ```
   Example? (only if the rule is non-obvious without one — empty to skip)
   ```

   Empty input → skip the Example, and that is the expected default. Only add an example when the rule genuinely needs an illustration to be understood. Never invent an Example for the user, and never pad with one just because the prompt asked.

Once you have rule + why (+ optional example), proceed to section assignment.

## Section assignment

Read the artifact's frontmatter to discover the available sections:

```bash
awk '/^---$/{i++; next} i==1' .claude/code-review/best-practices.md
```

Parse the YAML mentally. The available section list is the union of:

- The seven defaults (always present in any artifact written by `/init`): `security`, `bugs`, `performance`, `simplicity`, `testing`, `error_handling`, `conventions`.
- Any custom slugs present under the frontmatter `sections:` map.

The frontmatter is the source of truth — do not invent sections that aren't there. If the user wants a section that doesn't exist, follow the `<new>` flow below.

**Consistency check (structural).** Before walking sections, verify every frontmatter slug has a matching `## <Heading>` in the body. For each slug in the artifact's frontmatter `sections:` map: scan the body for `## ` headings, slugify each (lowercase, whitespace runs → `_`, drop non-`[a-z0-9_]`), and confirm at least one matches the slug. If a slug has no matching body heading, print:

```
The best-practices file is internally inconsistent: <slug> is missing in body.
Re-run /init (replace mode) or fix manually before continuing.
```

And exit. This check is **one-directional**: do NOT verify the reverse direction — body headings may include doc-only sections (like `## Rule format`) that don't appear in frontmatter.

Based on the rule text + Why, propose one section as the target.

If the artifact contains custom sections (any frontmatter slugs not in the seven defaults), include them in your proposal consideration. For each custom section, read its `## <Heading>` in the body and any existing rules to infer its scope, then propose accordingly. Custom sections are first-class candidates, not fallbacks.

For the seven defaults, use this rough mapping (your judgment overrides any single keyword):

- `security` — auth, secrets, injection, crypto, trust boundaries, input validation for adversarial input.
- `bugs` — null/undefined, off-by-one, race conditions, state correctness.
- `performance` — latency, throughput, allocations, query patterns, caching.
- `simplicity` — readability, indirection, naming, dead code.
- `testing` — coverage, fixtures, mocks, flaky tests, test structure.
- `error_handling` — exception flow, retries, logging, user-facing errors.
- `conventions` — formatting, naming, file layout, project-specific norms.

Print EXACTLY:

```
Target section: <proposed-slug>
Override? (enter slug, or <new> to create a new section, or empty to accept)
```

Parse the user's reply:

- Empty: accept the proposed slug.
- A slug already in the available list (case-insensitive match): use that slug.
- A slug NOT in the list: echo what was received and re-prompt:

  ```
  "<received>" is not an existing section. Available: <comma-separated list>. Use <new> to create one.
  Override? (enter slug, or <new> to create a new section, or empty to accept)
  ```
- The literal string `<new>` (case-insensitive): run the new-section flow.

### New-section flow

Prompt EXACTLY:

```
New section name?
```

Take the user's reply (whitespace-trimmed) as the display title. Slugify it to produce the frontmatter key:

- Lowercase the whole string.
- Replace any whitespace runs with a single underscore (`_`).
- Drop any character that is not `[a-z0-9_]`.

Verify the slug is non-empty and not already present in the artifact's frontmatter. On collision or empty result, echo and re-ask:

```
Slug "<slug>" is empty or already in use. Try again.
New section name?
```

Once the slug is accepted, you must update the artifact in two places **before** the rule is appended:

1. **Frontmatter:** add a new entry under `sections:`:

   ```yaml
     <slug>:
       refs: []
       model: inherit
   ```

   Insert it at the end of the `sections:` map (after the last existing section, before any blank line that separates `sections:` from `min_confidence:`).

2. **Body:** append a new heading at the bottom of the file (after the last `## ` section). Use the user's whitespace-trimmed reply verbatim as the display title — do **not** reconstruct it from the slug:

   ```markdown
   ## <Title>
   <!-- Add inline rules here, or rely on refs above -->
   ```

Use `Edit` for both. The body heading you just wrote is now the canonical display title for this slug — later steps will rediscover it by scanning the body, not by reconstructing from the slug. After this, the new slug is the target section and the placeholder comment is what's currently under it (so the rule will replace the placeholder per the rules in "Confirm and apply").

## Consistency check

Before any conflict scan or mutation, verify the artifact is internally consistent for the target slug:

1. The target section's slug exists in the frontmatter `sections:` map (or you just added it via the `<new>` flow).
2. The body contains a `## <Heading>` whose slugified form equals the target slug. To slugify a heading: take the text after `## `, lowercase it, replace whitespace runs with a single underscore (`_`), drop any character that is not `[a-z0-9_]`.

If either check fails, print EXACTLY:

```
The best-practices file is internally inconsistent: <slug> is <missing in frontmatter | missing in body>.
Re-run /init (replace mode) or fix manually before continuing.
```

And exit. Do not proceed with mutation.

## Conflict scan

Locate the body region for the target section by **scanning the body itself**, not by reconstructing the title from the slug:

1. Read the artifact body (everything after the closing `---` of the frontmatter).
2. Enumerate every line that starts with `## ` (exactly two hashes and a space).
3. For each such heading, slugify it as defined in the consistency check (lowercase, whitespace runs → `_`, drop non-`[a-z0-9_]`).
4. Pick the heading whose slugified form equals the target slug. That exact heading line is the anchor for the body region.

The body region runs from the line after the matched heading to the line before the next `## ` heading or EOF. The body is the source of truth for headings, so do not rely on a fixed slug→title map (e.g. `security`→`Security`); always discover the heading by slug-matching.

If no body heading slugifies to the target slug, the artifact is inconsistent — handle per the consistency check above (print the inconsistency message and exit).

Identify existing rule bullets — top-level lines starting with `- **`. Ignore the placeholder comment (`<!-- Add inline rules here, or rely on refs above -->`) and any sub-bullets (`  - **Why:**`, `  - **Example:**`, `  - **Source:**`).

For each existing rule bullet, compare semantically against the new rule (use your judgment based on the rule text + Why; do not rely on a fixed metric).

**Similarity anchor.** Surface a match if the existing rule covers the same **imperative subject** as the new rule — i.e. they share the primary verb + the primary noun phrase. Examples: "always validate input at trust boundaries" and "check user input at API entry points" are matches (verb: validate/check, noun phrase: input at boundaries). "always check for null" and "always validate input" are not matches (different noun phrases). Wording differences alone don't count; the underlying rule subject must overlap.

Surface up to **three** closest matches.

If at least one similar rule is found, print EXACTLY:

```
Found similar rule(s):
  L<n>: <bullet text> ...
  L<n>: <bullet text> ...
  1) Replace
  2) Merge
  3) Keep both  [recommended]
  4) Skip
Choose [3]:
```

Where `<n>` is the line number of the matched bullet in the artifact and `<bullet text>` is the bare rule sentence (truncate to ~80 chars with `...` if longer). Show one line per match, up to three.

Parse the reply as a single digit; empty defaults to `3`. Re-prompt with `Enter a number 1-4.` followed by the same options block on any other input.

- `1` (replace): edit the matched bullet **in place** so its rule text becomes the new rule, and replace **all** its sub-bullets — every indented `- **…:**` line (Why, Example, Source) — with the new ones. The new bullet has Why (mandatory) and Example (only if the user provided one). The discussion-flow does not generate Source; if the matched bullet had a `**Source:**` from a prior PR-flow write, it is dropped (the rewrite invalidates that citation). If multiple matches were surfaced, ask the user which line to replace before editing — do not auto-pick.
- `2` (merge): ask the user to confirm a merged single-rule text. Print:

  ```
  Proposed merged rule: <merged text>
  Confirm? (y to accept, or type a replacement)
  ```

  Use the user's confirmed text as the new rule sentence. Then edit the matched bullet in place using the same whole-block replacement semantics as `1` (replace). If multiple matches were surfaced, ask which line to merge into.
- `3` (keep both, default): append the new rule as a new bullet at the end of the section.
- `4` (skip): bail out without writing anything. Print `Skipped — no change.` and jump to the loop step.

If no similar rules are found, skip the conflict prompt and proceed straight to "Confirm and apply" with append-as-new-bullet semantics.

**Never auto-merge.** **Never silently rewrite an existing rule** — always confirm replace/merge with the user before editing.

## Confirm and apply

Render the lines that will be added or changed in a fenced code block. The block must show exactly the new rule bullet (and any rewritten existing bullet, in the case of replace/merge), formatted in the canonical rule format:

```markdown
- **<rule sentence>**
  - **Why:** <reason>
  - **Example:** <example>
```

When the user skipped the example, omit the `**Example:**` sub-bullet entirely (do not write `**Example:** N/A` or similar).

Then print EXACTLY:

```
Apply? (y)es / (e)dit / (n)o
```

Parse the reply (case-insensitive single character; empty defaults to `n`):

- `y`: apply the change.
- `e`: enter inline-edit mode. Ask which field the user wants to revise:

  ```
  Edit which?
    1) Rule
    2) Why
    3) Example
    4) All
  Choose:
  ```

  Parse the reply as a single digit `1-4`; this prompt has no default — re-prompt with `Enter a number 1-4.` followed by the same options block on empty or invalid input. Then re-prompt for the chosen field(s), update your in-memory state, re-render the diff and re-ask `Apply? (y)es / (e)dit / (n)o`. Loop until the user picks `y` or `n`.
- `n`: bail out without writing. Print `Aborted — no change.` and jump to the loop step.

On `y`, use `Edit` to apply the change to `.claude/code-review/best-practices.md`. Anchor the body region the same way as in the conflict scan: scan the body for `## ` headings, slugify each, and use the heading whose slugified form equals the target slug as the anchor — do not reconstruct the title from the slug. The semantics depend on the conflict-scan outcome:

- **Replace** or **merge**: locate the matched bullet and **all** its sub-bullets — every indented `- **…:**` line immediately following (Why, Example, Source) — and replace that whole bullet block with the new bullet block. The discussion-flow's new bullet block contains Rule + Why and an optional Example; it never contains a Source. Any pre-existing `**Source:**` on the matched bullet is dropped on replace/merge.
- **Keep both / no conflict (append)**: insert the new bullet at the end of the section's body region — i.e. immediately before the next `## ` heading, or before EOF if the target section is the last section.
  - **Placeholder rule:** if the section currently contains only the placeholder comment `<!-- Add inline rules here, or rely on refs above -->` (and nothing else), **replace** the placeholder line with the new bullet block. The placeholder must be removed once a section has at least one rule.
  - **Otherwise:** keep the existing rules in place and append the new bullet block after the last existing rule. Do not touch the placeholder if it is somehow present alongside other rules (it should not be — but if it is, leave it for the user to clean up).
- **New-section flow**: the new section was just created with the placeholder; the placeholder-rule above applies — replace it with the new bullet block.

Preserve the placeholder comment in **all sections you are not modifying**. Only the section the new rule lands in loses its placeholder (and only if the section had nothing else).

## Loop

After a successful write, print EXACTLY:

```
Added 1 rule to <section>.
```

Where `<section>` is the slug the rule was written under.

Then prompt EXACTLY:

```
Another? (y/n) [n]
```

Parse the reply (case-insensitive single character; empty defaults to `n`):

- `y`: jump back to the **Topic seed** step (re-prompt `What rule do you want to formalize?` regardless of the original `$ARGUMENTS` — the seed only seeds the first iteration).
- `n` or empty: exit silently.

If the user bailed earlier (no Why provided, skip in conflict scan, `n` at confirm), still ask `Another? (y/n) [n]` so they can recover from a wrong answer without re-invoking the command.

## Hard rules

These constraints are non-negotiable. Re-read them before each step:

- **Never write a rule without a `**Why:**`.** If the user cannot articulate one, skip the rule. Do not invent rationales.
- **Never invent a Why or Example for the user.** Both come from the user's words. Optional means optional — empty Example skips the sub-bullet entirely.
- **Never silently rewrite existing rules.** Always run the conflict scan and surface matches. The user must explicitly choose `replace` or `merge` before any existing bullet is edited.
- **Never auto-merge.** Merge requires user confirmation of the merged text.
- **Preserve the placeholder comment** (`<!-- Add inline rules here, or rely on refs above -->`) in all sections you are not modifying. Only the section the new rule lands in loses its placeholder, and only if it had nothing else.
- **The artifact frontmatter is the source of truth for the section list.** Do not invent sections that aren't there. If the user wants a new section, run the `<new>` flow — it adds the slug to frontmatter (`refs: []`, `model: inherit`) AND a `## <Title>` heading to the body before the rule is written.
- **Rule format must match Phase 2 template:**

  ```markdown
  - **<rule sentence>**
    - **Why:** <reason>
    - **Example:** <example>
  ```

  Omit the `**Example:**` sub-bullet entirely when the example is skipped. The discussion-flow never emits a `**Source:**` sub-bullet; that field is reserved for `/gather-insight-pr` citations.
- **Skip the example unless the rule is non-obvious without one.** Empty input is the expected default. Do not invent or pad an example just because the prompt asked.
- **Render numbered prompts exactly as specified** — header, indented options, `[recommended]` marker on the recommended option, then a `Choose [N]:` line (or bare `Choose:` when there is no default). Accept only digits in `1..N` or empty (when a default exists). On any other input, re-prompt with `Enter a number 1-N.` followed by the same options block. Do not accept single-letter shortcuts.
- **Use only the declared tools:** `Bash`, `Read`, `Edit`, `Write`, `Glob`, `Grep`. Do not request anything else.
