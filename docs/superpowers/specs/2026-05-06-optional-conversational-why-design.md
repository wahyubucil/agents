# Optional, Conversational "Why" in Review Comments — Design

## Goal

Stop forcing every review comment posted by `/review` and `/review-discussion` to include a "why" paragraph, and ban citation-style framing that names specific source artifacts (e.g. `AGENTS.md §2 mandates...`, `matches path:Lx-Ly precedent`, `violates the <section-slug> rule X`).

The change flips the default: a comment is just `<message>` + the anchored code unless the agent decides a short, conversational explanation actually adds value.

## Motivation

The current style-rules block (added in `739ce9b feat(code-review): tighten /review-discussion schema + add comment-style rules` and the parallel commit on `/review`) tells the agent to *Lead with WHY in 1–3 short sentences, grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly` precedent", "violates the <section-slug> rule X")*. In practice this produces comments like:

> `AGENTS.md §2` mandates "Concurrency: Kotlin Coroutines & Flow", and the project already provides an injectable `@ApplicationScope CoroutineScope` in `CoroutinesModule.kt`...

Two problems:

1. **Tool-coupled framing.** Citing `AGENTS.md` / `CLAUDE.md` / specific section numbers assumes every team using the plugin keeps their convention doc in the same place under the same name. They don't.
2. **Forced verbosity.** Many findings (rename, flip a condition, remove an unused arg) are self-explanatory once `message` and the anchored code are in front of the reviewer. Forcing a "why" paragraph adds noise.

## Scope

Markdown-only edits to two files:

- `plugins/code-review/commands/review.md`
- `plugins/code-review/commands/review-discussion.md`

No code changes, no new files, no test framework. These are LLM agent prompts; verification is `grep` for stale fragments, file re-reads for internal consistency, and an optional manual smoke test on a real PR.

## Out of scope

- Changing the posting CLI (`gh-pr-review`) or its flags.
- Changing the merge / filter / posting flow beyond the two specific spots called out below.
- Reworking the `🤖 _Agent review — <sections>_` / `👤 _Human review, helped by agent_` footer markers (those just shipped in `0.3.0` and stay as-is).
- Touching `templates/best-practices.template.md` — it doesn't dictate comment style.

## Design

### 1. Schema: `evidence` is optional

Both commands embed a YAML schema in the agent prompt. The `evidence` field comment changes to make optionality explicit.

`review.md` step 7 (parallel-agent prompt schema):

```yaml
findings:
  - path: <relative path>
    line_start: <int>
    line_end: <int>
    message: <imperative one-liner — what to do>
    evidence: <optional body of the comment — see style rules below; omit/empty when not needed>
    confidence: <0-100>
```

`review-discussion.md` `suggested_findings` schema: same `evidence` field comment.

The `If no findings, return findings: []` follow-up sentence stays untouched. Each command file already tells the agent how it expresses absence at the *list* level; this change is purely about absence at the *individual `evidence`* level.

### 2. Comment style rules: rewrite the WHY guidance

The current style-rules block has two bullets that need replacement:

- The first bullet currently says the body "just adds a one-line why — do not leave it empty."
- The second bullet currently says "Lead with WHY in 1–3 short sentences, grounded inline in the rule or code reference (e.g. 'matches `path:Lx-Ly` precedent', 'violates the <section-slug> rule X')."

These two bullets are replaced with the following three bullets (verbatim, copy-paste into both files; whitespace differs because `review.md` indents the block 5 spaces under a numbered step and `review-discussion.md` indents it 7 spaces under a nested bullet):

```
- `message` is imperative and short. State what to do.

- `evidence` is OPTIONAL. Leave it empty when `message` plus the anchored
  code already makes the change obvious. Most simple fixes (rename, flip
  a condition, remove an unused arg, reorder two lines) need no body. Only
  add a body when the WHY isn't obvious from message + code alone.

- When you do write a body, keep it conversational and simple (1–3 short
  sentences). Talk about concrete impact or the convention plainly. Do NOT
  cite specific files or sections — no "AGENTS.md §2 mandates...", no
  "matches `path:Lx-Ly` precedent", no "violates the <section-slug> rule X".
  Write the way a senior reviewer would in a quick comment, not the way a
  static analyzer surfaces a rule ID. For complex fixes, include a fenced
  code block showing the concrete change.
```

The remaining bullets in the style-rules block stay verbatim:

- "Do not repeat the message in the body. Do not list multiple separate angles. Do not append a separate 'Evidence:' paragraph — citations live inline in the prose, not as a footer."
- The "When the fix is a literal in-place replacement..." `suggestion`-vs-plain-block bullet (with the three preconditions).
- "One suggestion block per comment. GitHub only applies the first."

### 3. Body format: handle empty evidence

When `evidence` is empty/null, the posted comment collapses. Both posting heredocs already template `<message>` then `<evidence>` then a separator and footer; the agent needs an explicit rule for the empty case so it doesn't emit a stray blank paragraph.

**`review.md`** — body-format prose update:

- The `<evidence> paragraph(s)` bullet gains a parenthetical: *"...if non-empty. If `evidence` is empty/null, omit this paragraph entirely — the body is just `<message>` then the separator and footer."*
- The heredoc template stays the same; the change is in the human-readable body-format prose immediately below it.

**`review-discussion.md`** — same body-format prose update on its mirror bullet.

The heredocs themselves are not changed — they already use `<evidence>` as a placeholder, and "place an empty string here" is a valid agent interpretation once the prose says so.

### 4. Merge step: skip empty evidence in concatenation

`review.md` Merge section currently says: *"Concatenate evidence with `\n\n` (a blank line between paragraphs), in confidence-descending order."*

Updated to: *"Concatenate **non-empty** evidence strings with `\n\n` (a blank line between paragraphs), in confidence-descending order. If every contributor's `evidence` is empty, the merged finding's `evidence` is empty too."*

The "One code block per merged comment" rule below it stays verbatim — it already keys off "if two or more contributing findings each include a fenced code block", which gracefully handles the empty case (zero fenced code blocks → no stripping needed).

`review-discussion.md` has no merge step (single-agent flow), so no change there.

### 5. Cross-file consistency check

The two command files must keep their substantive style-rules body identical word-for-word, just like the prior plan's Task 6 enforced. The verification is the same `diff` of the trimmed block:

```bash
diff <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review.md | sed 's/^[[:space:]]*//') \
     <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review-discussion.md | sed 's/^[[:space:]]*//')
```

Expected: empty output.

## Architecture

No architecture change. The plugin already has:

- A schema that section agents (and the discussion agent) emit findings against.
- A merge step that groups overlapping findings and concatenates evidence.
- A posting step that templates a comment body from `<message>` and `<evidence>` with a footer.

This change adjusts the *content rules* embedded in those steps, not the steps themselves.

## Data flow

`/review` flow (no structural change):

```
section agents → flat finding list → merge (group overlap, concat non-empty
evidence) → filter by min_confidence → post each finding as <message> + (optional)
<evidence> + --- + 🤖 footer
```

`/review-discussion` flow (no structural change):

```
single answering agent → optional suggested_findings → user picks which to post →
post each as <message> + (optional) <evidence> + --- + 👤 footer
```

The empty-evidence path is the only new branch in the templating logic, and it's a pure prose rule for the agent — no code branch, no template branch.

## Error handling

No new error modes. Existing failure paths stay:

- Anchor outside the in-scope diff → log `failed to anchor: <path>:<start>-<end>`, continue.
- `gh pr-review --start` failure → surface the error, exit (user re-runs).
- Empty findings list after filter → still post pending review with summary body that says "no high-confidence findings."

If the agent emits a finding with both empty `message` and empty `evidence`, that's an upstream agent error — the schema already requires `message` as non-empty (imperative one-liner), and there's no mechanism to bypass that. We don't add new validation; we trust the schema.

## Testing

This is a markdown-only edit to LLM-agent prompts. No unit tests apply. Verification is in three layers:

**Static (mandatory):**

- `grep` for stale fragments to confirm they're gone:
  - `grep -n "Lead with WHY" plugins/code-review/commands/` → 0 hits.
  - `grep -n "matches \`path:Lx-Ly\` precedent" plugins/code-review/commands/` → 0 hits.
  - `grep -n "violates the <section-slug> rule X" plugins/code-review/commands/` → 0 hits.
- File re-reads to confirm new bullets are in the right place and the schema field comment matches the spec.
- The cross-file `diff` from Section 5.

**Manual smoke test (recommended):**

Run `/review` against a small open PR with a known issue (or a throwaway PR). Confirm:

1. At least one posted comment is just `<message>` + anchored code (no body) — proving "evidence is optional" landed.
2. At least one posted comment that *does* have a body reads conversationally and contains zero `AGENTS.md §`, `path:Lx-Ly precedent`, or `<section-slug> rule X` framing.
3. The footer `🤖 _Agent review — <sections>_` is unchanged.

Run `/review-discussion <pr> <question>` against a PR with a question that surfaces at least one suggested finding. Same checks as above against the `👤 _Human review, helped by agent_` footer.

If the agent regresses to citing source artifacts in the body, capture the comment text and tighten the wording before re-shipping — don't paper over with a follow-up rule.

## Risks and mitigations

- **Risk:** Agent omits a "why" on a finding where one was genuinely warranted.
  **Mitigation:** Style rule explicitly says *"Only add a body when the WHY isn't obvious from message + code alone."* The default flips, but the door stays open. If smoke testing shows over-omission on subtle findings, follow up with a tighter "when in doubt, add a one-line why" rule.

- **Risk:** Agent finds a creative way to leak source citations (e.g. inline `(per project convention X)` parenthetical).
  **Mitigation:** The bullet bans three specific patterns by example. If a new pattern surfaces during smoke testing, add it to the banned-examples list rather than rewriting the rule.

- **Risk:** Drift between `review.md` and `review-discussion.md` — the two style-rule blocks could diverge on a future edit.
  **Mitigation:** Section 5's `diff` check is part of the implementation plan and any future plan that touches either block. Both files will continue to inline the block verbatim — no shared partial — because the plugin has no template system and inlining keeps each command independently readable (same rationale as the prior plan).

## Migration / rollout

No migration. The next `/review` invocation after merge picks up the new prompts. Existing pending reviews on GitHub are untouched.

## File list

| File | Change |
| --- | --- |
| `plugins/code-review/commands/review.md` | Schema field comment, two style-rule bullets replaced (one new bullet inserted), body-format prose, merge wording. |
| `plugins/code-review/commands/review-discussion.md` | Schema field comment, two style-rule bullets replaced (one new bullet inserted), body-format prose. |
| `plugins/code-review/.claude-plugin/plugin.json` | Version bump (`0.3.0` → `0.4.0`) — comment-format change is user-visible. |

## Versioning

Plugin currently sits at `0.3.0` (per `f182fb1 feat(code-review): numbered choice prompts + Source citation label, bump 0.3.0`). Bump to `0.4.0` — the visible behavior of every posted comment changes, which is a noteworthy step for users running the installed plugin.
