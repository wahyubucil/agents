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

3. **Example** (optional): a short code or text snippet illustrating the rule. Ask once:

   ```
   Example? (code/text snippet, or empty to skip)
   ```

   Empty input → skip the Example. Never invent an Example for the user.

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

Based on the rule text + Why, propose one section as the target. Use this rough mapping (your judgment overrides any single keyword):

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

Take the user's reply (whitespace-trimmed) as the display title. Slugify it:

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

2. **Body:** append a new heading at the bottom of the file (after the last `## ` section):

   ```markdown
   ## <Title>
   <!-- Add inline rules here, or rely on refs above -->
   ```

Use `Edit` for both. After this, the new slug is the target section and the placeholder comment is what's currently under it (so the rule will replace the placeholder per the rules in "Confirm and apply").

## Conflict scan

Read the artifact and locate the body region under the target section heading (`## <Title>` derived from the slug — defaults map: `security`→`Security`, `bugs`→`Bugs`, `performance`→`Performance`, `simplicity`→`Simplicity`, `testing`→`Testing`, `error_handling`→`Error Handling`, `conventions`→`Conventions`; for custom sections use the title recorded when the section was created). The body region runs from the line after the heading to the line before the next `## ` heading or EOF.

Identify existing rule bullets — top-level lines starting with `- **`. Ignore the placeholder comment (`<!-- Add inline rules here, or rely on refs above -->`) and any sub-bullets (`  - **Why:**`, `  - **Example:**`).

For each existing rule bullet, compare semantically against the new rule (use your judgment based on the rule text + Why; do not rely on a fixed metric). Surface up to **three** closest matches.

If at least one similar rule is found, print EXACTLY:

```
Found similar rule(s):
  L<n>: <bullet text> ...
  L<n>: <bullet text> ...
(r)eplace / (m)erge / (k)eep both / (s)kip [k]
```

Where `<n>` is the line number of the matched bullet in the artifact and `<bullet text>` is the bare rule sentence (truncate to ~80 chars with `...` if longer). Show one line per match, up to three.

Parse the reply (case-insensitive single character; empty defaults to `k`):

- `r` (replace): edit the matched bullet **in place** so its rule text becomes the new rule, and replace its `**Why:**` and `**Example:**` sub-bullets with the new ones (drop the example sub-bullet entirely if the new rule has no example). If multiple matches were surfaced, ask the user which line to replace before editing — do not auto-pick.
- `m` (merge): ask the user to confirm a merged single-rule text. Print:

  ```
  Proposed merged rule: <merged text>
  Confirm? (y to accept, or type a replacement)
  ```

  Use the user's confirmed text as the new rule sentence. Then edit the matched bullet in place, replacing rule + Why + Example. If multiple matches were surfaced, ask which line to merge into.
- `k` (keep both, default): append the new rule as a new bullet at the end of the section.
- `s` (skip): bail out without writing anything. Print `Skipped — no change.` and jump to the loop step.

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
  Edit which? (r)ule / (w)hy / (e)xample / (a)ll
  ```

  Re-prompt for the chosen field(s), update your in-memory state, then re-render the diff and re-ask `Apply? (y)es / (e)dit / (n)o`. Loop until the user picks `y` or `n`.
- `n`: bail out without writing. Print `Aborted — no change.` and jump to the loop step.

On `y`, use `Edit` to apply the change to `.claude/code-review/best-practices.md`. The semantics depend on the conflict-scan outcome:

- **Replace** or **merge**: locate the matched bullet (and its `**Why:**`/`**Example:**` sub-bullets, which are the indented lines immediately following) and replace that whole bullet block with the new bullet block.
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

  Omit the `**Example:**` sub-bullet entirely when the example is skipped.
- **Use only the declared tools:** `Bash`, `Read`, `Edit`, `Write`, `Glob`, `Grep`. Do not request anything else.
