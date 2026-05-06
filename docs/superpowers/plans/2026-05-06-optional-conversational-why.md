# Optional, Conversational "Why" Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `evidence` optional in review-comment schemas, ban citation-style framing (`AGENTS.md §X mandates...`, `matches path:Lx-Ly precedent`, `violates the <section-slug> rule X`), and rewrite the comment-style rules so the agent only adds a body when the WHY isn't obvious from `message` + anchored code.

**Architecture:** Edit-only change against two slash-command markdown files (`plugins/code-review/commands/review.md`, `plugins/code-review/commands/review-discussion.md`) and the plugin's `plugin.json` for a version bump. No new files, no test framework — these markdown files are LLM agent prompts. Verification uses `grep` for stale fragments, file re-reads for internal consistency, and an optional manual smoke test on a real PR.

**Tech Stack:** Markdown command files; `gh-pr-review` CLI extension (unchanged); GitHub PR review API (unchanged).

**Spec:** `docs/superpowers/specs/2026-05-06-optional-conversational-why-design.md`

---

## File Structure

Files modified by this plan:

- `plugins/code-review/commands/review.md` — schema field comment (~line 308), comment-style rules block (~lines 314–346), merge step wording (~line 356), body-format prose (~line 429).
- `plugins/code-review/commands/review-discussion.md` — schema field comment (~line 326), comment-style rules block (~lines 332–364), body-format prose (~lines 438–439).
- `plugins/code-review/.claude-plugin/plugin.json` — version bump `0.3.0` → `0.4.0`.

No new files. The style-rules block stays inlined verbatim in both command files (no shared partial — same rationale as the prior natural-comment-format plan: no template system, and inlining keeps each command independently readable).

A note about line numbers: edits in earlier tasks shift the line numbers cited in later tasks. Use the anchor strings shown in each task's Edit step to locate the target — do not rely on line numbers across tasks.

---

### Task 1: Update schema and rewrite comment-style rules in `review.md`

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the step 8 `Output schema instruction` block and the step 9 `Comment style rules (verbatim):` block.

- [ ] **Step 1: Open the file and locate the schema block.**

Run: `grep -n "Output schema instruction" plugins/code-review/commands/review.md`
Expected: one hit, around line 300.

- [ ] **Step 2: Update the schema's `evidence` field comment.**

Use the `Edit` tool. Match the existing block exactly (5-space indent under the numbered list step).

`old_string`:

```
  8. **Output schema instruction:**

     ```yaml
     findings:
       - path: <relative path>
         line_start: <int>
         line_end: <int>
         message: <imperative one-liner — what to do>
         evidence: <the body of the comment — see style rules below>
         confidence: <0-100>
     ```

     Followed by: `If no findings, return findings: [].`
```

`new_string`:

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
```

- [ ] **Step 3: Locate the comment-style rules block.**

Run: `grep -n "Comment style rules (verbatim):" plugins/code-review/commands/review.md`
Expected: one hit, around line 314.

- [ ] **Step 4: Replace the first two style-rule bullets with three new bullets.**

Use the `Edit` tool. The block sits inside numbered step 9 with 5-space indent; bullets inside the fenced block are indented to match (the `- ` lines start with 5 spaces).

`old_string`:

```
     - `message` is imperative and short. For simple fixes (rename, flip a condition,
       remove an unused arg, reorder two lines), the message alone should be enough;
       the body just adds a one-line why — do not leave it empty.

     - `evidence` is the body of the comment. Lead with WHY in 1–3 short sentences,
       grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly`
       precedent", "violates the <section-slug> rule X"). For complex fixes, include
       a fenced code block showing the concrete change.
```

`new_string`:

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
```

- [ ] **Step 5: Verify by re-reading.**

Read `plugins/code-review/commands/review.md` lines 295–360. Confirm:
  - The schema's `evidence` field comment now reads `<optional body of the comment — see style rules below; omit/empty when not needed>`.
  - The style-rules block now contains three new bullets (the `message` short bullet, the optional-`evidence` bullet, and the conversational-WHY bullet).
  - The remaining bullets ("Do not repeat the message in the body...", "When the fix is a literal in-place replacement...", "One suggestion block per comment...") are still present and unchanged.

Run: `grep -n "Lead with WHY" plugins/code-review/commands/review.md`
Expected: zero hits.

Run: `grep -n "matches \`path:Lx-Ly\` precedent" plugins/code-review/commands/review.md`
Expected: zero hits in non-banned-example contexts. The phrase only survives inside the new bullet's banned-examples list (one hit).

Run: `grep -n "do not leave it empty" plugins/code-review/commands/review.md`
Expected: zero hits.

- [ ] **Step 6: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): make /review evidence optional + ban source citations

evidence is now optional in the schema and style rules; comments default to
just message + anchored code unless the WHY is non-obvious. Bans the
"AGENTS.md §X mandates..." / "matches path:Lx-Ly precedent" / "violates the
<section-slug> rule X" framing that leaked tool-coupled phrasing into bodies.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Update merge step in `review.md` to skip empty evidence

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the `## Merge` section's "Concatenate evidence" bullet.

- [ ] **Step 1: Locate the merge bullet.**

Run: `grep -n "Concatenate evidence with" plugins/code-review/commands/review.md`
Expected: exactly one hit, inside the `## Merge` section.

- [ ] **Step 2: Update the bullet to skip empty evidence.**

Use the `Edit` tool.

`old_string`:

```
- **Concatenate evidence with `\n\n`** (a blank line between paragraphs), in confidence-descending order. Inline ` | ` joins read awkwardly when evidence contains multi-sentence prose or fenced code blocks; double-newline joins read like consecutive paragraphs.
```

`new_string`:

```
- **Concatenate non-empty evidence strings with `\n\n`** (a blank line between paragraphs), in confidence-descending order. Skip contributors whose `evidence` is empty/null — they contributed only their section slug to `contributing_sections`. If every contributor's `evidence` is empty, the merged finding's `evidence` is empty too. Inline ` | ` joins read awkwardly when evidence contains multi-sentence prose or fenced code blocks; double-newline joins read like consecutive paragraphs.
```

- [ ] **Step 3: Verify by re-reading.**

Read the `## Merge` section end-to-end. Confirm:
  - The "Concatenate" bullet now starts with **non-empty** in bold.
  - The bullet documents the empty-input and all-empty cases.
  - The "One code block per merged comment" bullet immediately below is unchanged.

Run: `grep -n "Concatenate evidence with" plugins/code-review/commands/review.md`
Expected: zero hits (replaced by "Concatenate non-empty evidence strings with").

Run: `grep -n "Concatenate non-empty evidence" plugins/code-review/commands/review.md`
Expected: exactly one hit.

- [ ] **Step 4: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): merge step skips empty evidence in concatenation

Now that evidence is optional, the Merge step concatenates only non-empty
evidence strings. If every contributor in a group leaves evidence empty,
the merged finding's evidence stays empty (body collapses on post).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Update body-format prose in `review.md` to handle empty evidence

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the `**Body format (must follow exactly):**` bullets that document the posted comment shape.

- [ ] **Step 1: Locate the body-format prose.**

Run: `grep -n "Body format (must follow exactly)" plugins/code-review/commands/review.md`
Expected: exactly one hit.

- [ ] **Step 2: Update the `<evidence>` paragraph bullet.**

Use the `Edit` tool. Match the existing bullet exactly (no leading spaces, top-level list).

`old_string`:

```
- `<message>` line: the finding's `message` field — an imperative one-liner.
- Blank line.
- `<evidence>` paragraph(s): the merged evidence string from step 8 (paragraphs joined by blank lines; may include a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise).
- Blank line, then the literal separator `---` on its own line.
- Footer line: `🤖 _Agent review — <sections>_` where `<sections>` is the merged finding's `contributing_sections` list joined with ` · ` (space–middle-dot–space). Slugs render verbatim — lowercase, underscores intact, no title-casing. For a single-section finding this is just the one slug (e.g. `🤖 _Agent review — bugs_`).
```

`new_string`:

```
- `<message>` line: the finding's `message` field — an imperative one-liner.
- Blank line.
- `<evidence>` paragraph(s): the merged evidence string from step 8 (paragraphs joined by blank lines; may include a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise). **If `evidence` is empty/null, omit this paragraph and the blank line above it entirely** — the body collapses to `<message>`, the separator, and the footer.
- Blank line, then the literal separator `---` on its own line.
- Footer line: `🤖 _Agent review — <sections>_` where `<sections>` is the merged finding's `contributing_sections` list joined with ` · ` (space–middle-dot–space). Slugs render verbatim — lowercase, underscores intact, no title-casing. For a single-section finding this is just the one slug (e.g. `🤖 _Agent review — bugs_`).
```

- [ ] **Step 3: Verify by re-reading.**

Read the `**Body format (must follow exactly):**` bullets through the next `**Anchoring rules:**` heading. Confirm:
  - The `<evidence>` bullet now contains the bolded sentence about omitting the paragraph when evidence is empty/null.
  - The other four bullets are unchanged.
  - The `**Anchoring rules:**` block immediately below is unchanged.

Run: `grep -n "If \`evidence\` is empty/null, omit this paragraph" plugins/code-review/commands/review.md`
Expected: exactly one hit.

- [ ] **Step 4: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): /review body collapses to message-only when evidence empty

The body-format prose now tells the agent to omit the evidence paragraph
(and its leading blank line) when the merged evidence is empty. Body shape
becomes: <message> -> --- -> 🤖 footer. Heredoc template is unchanged.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Update schema and rewrite comment-style rules in `review-discussion.md`

**Files:**

- Modify: `plugins/code-review/commands/review-discussion.md` — the `suggested_findings` schema block and the `Comment style rules` bullet inside the answering-agent prompt.

- [ ] **Step 1: Locate the schema block.**

Run: `grep -n "suggested_findings:" plugins/code-review/commands/review-discussion.md`
Expected: at least one hit inside a YAML schema block in the "Answering instructions" step.

- [ ] **Step 2: Update the schema's `evidence` field comment.**

Use the `Edit` tool. The schema sits inside a nested-list step; the bullet is indented 5 spaces and the YAML block inside it is indented 7 spaces.

`old_string`:

```
       ```yaml
       suggested_findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <imperative one-liner — what to do>
           evidence: <the body of the comment — see style rules below>
           confidence: <0-100>
       ```
```

`new_string`:

```
       ```yaml
       suggested_findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <imperative one-liner — what to do>
           evidence: <optional body of the comment — see style rules below; omit/empty when not needed>
           confidence: <0-100>
       ```
```

- [ ] **Step 3: Locate the style-rules block.**

Run: `grep -n "Comment style rules (apply to every entry" plugins/code-review/commands/review-discussion.md`
Expected: exactly one hit.

- [ ] **Step 4: Replace the first two style-rule bullets with three new bullets.**

Use the `Edit` tool. The block sits inside a sub-bullet of a numbered step with 7-space indent; the inner bullets are indented 7 spaces too. Match the existing whitespace exactly.

`old_string`:

```
       - `message` is imperative and short. For simple fixes (rename, flip a condition,
         remove an unused arg, reorder two lines), the message alone should be enough;
         the body just adds a one-line why — do not leave it empty.

       - `evidence` is the body of the comment. Lead with WHY in 1–3 short sentences,
         grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly`
         precedent", "violates the <section-slug> rule X"). For complex fixes, include
         a fenced code block showing the concrete change.
```

`new_string`:

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
```

- [ ] **Step 5: Verify by re-reading.**

Read the surrounding section (the "Answering instructions" step) end-to-end. Confirm:
  - The schema's `evidence` field comment now reads `<optional body of the comment — see style rules below; omit/empty when not needed>`.
  - The style-rules block contains three new bullets (the `message` short bullet, the optional-`evidence` bullet, and the conversational-WHY bullet).
  - The remaining bullets in the block ("Do not repeat the message in the body...", "When the fix is a literal in-place replacement...", "One suggestion block per comment...") are unchanged.
  - Sibling bullets above and below the style-rules sub-bullet are unchanged.

Run: `grep -n "Lead with WHY" plugins/code-review/commands/review-discussion.md`
Expected: zero hits.

Run: `grep -n "do not leave it empty" plugins/code-review/commands/review-discussion.md`
Expected: zero hits.

- [ ] **Step 6: Commit.**

```bash
git add plugins/code-review/commands/review-discussion.md
git commit -m "$(cat <<'EOF'
feat(code-review): make /review-discussion evidence optional + ban citations

Mirror the /review change in /review-discussion: evidence is optional in the
suggested_findings schema, and the style-rules block bans tool-coupled
framing ("AGENTS.md §X mandates...", path:Lx-Ly precedent, <section-slug>
rule X) in favor of conversational WHY when one is genuinely needed.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Update body-format prose in `review-discussion.md` to handle empty evidence

**Files:**

- Modify: `plugins/code-review/commands/review-discussion.md` — the body-format bullets that document the posted comment shape (~line 437).

- [ ] **Step 1: Locate the body-format bullets.**

Run: `grep -n "👤 _Human review, helped by agent_" plugins/code-review/commands/review-discussion.md`
Expected: two hits — one in the heredoc, one in the body-format bullets that immediately follow.

- [ ] **Step 2: Update the `<evidence>` paragraph bullet.**

Use the `Edit` tool. The bullets are indented 3 spaces under their numbered step.

`old_string`:

```
   - `<message>` line: the finding's `message` field — an imperative one-liner.
   - Blank line, then `<evidence>` paragraph(s): the finding's `evidence` field (may contain a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise).
   - Blank line, then the literal separator `---` on its own line.
   - Footer line: literally `👤 _Human review, helped by agent_`. This is fixed text — `/review-discussion` always runs as a single agent answering one question, so there are no per-section variations.
```

`new_string`:

```
   - `<message>` line: the finding's `message` field — an imperative one-liner.
   - Blank line, then `<evidence>` paragraph(s): the finding's `evidence` field (may contain a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise). **If `evidence` is empty/null, omit this paragraph and the blank line above it entirely** — the body collapses to `<message>`, the separator, and the footer.
   - Blank line, then the literal separator `---` on its own line.
   - Footer line: literally `👤 _Human review, helped by agent_`. This is fixed text — `/review-discussion` always runs as a single agent answering one question, so there are no per-section variations.
```

- [ ] **Step 3: Verify by re-reading.**

Read the surrounding posting-flow section end-to-end. Confirm:
  - The `<evidence>` bullet now contains the bolded sentence about omitting the paragraph when evidence is empty/null.
  - The other three bullets are unchanged.
  - The bullets that follow ("If `line_end == line_start`, omit `--start-line`...", "Anchors must be inside the diff...") are unchanged.

Run: `grep -n "If \`evidence\` is empty/null, omit this paragraph" plugins/code-review/commands/review-discussion.md`
Expected: exactly one hit.

- [ ] **Step 4: Commit.**

```bash
git add plugins/code-review/commands/review-discussion.md
git commit -m "$(cat <<'EOF'
feat(code-review): /review-discussion body collapses when evidence empty

Body-format bullets now tell the agent to omit the evidence paragraph (and
its leading blank line) when evidence is empty. Body shape becomes:
<message> -> --- -> 👤 footer. Heredoc template is unchanged.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Cross-file consistency pass + version bump

**Files:**

- Read-only verification across `plugins/code-review/commands/` for substantive style-rule alignment.
- Modify: `plugins/code-review/.claude-plugin/plugin.json` — version bump.

- [ ] **Step 1: Confirm no leftover stale fragments anywhere in the plugin commands.**

Run: `grep -rn "Lead with WHY\|do not leave it empty" plugins/code-review/commands/`
Expected: zero hits.

Run: `grep -rn "matches \`path:Lx-Ly\` precedent\|violates the <section-slug> rule X" plugins/code-review/commands/`
Expected: each phrase appears exactly twice — once each in `review.md` and `review-discussion.md`, both inside the new "Do NOT cite specific files or sections" banned-examples list. Two files × one occurrence each = two total per phrase. If you see them anywhere else (e.g. an old bullet that should have been replaced), edit the offending file.

Run: `grep -rn "AGENTS.md §2 mandates" plugins/code-review/commands/`
Expected: exactly two hits — one in each command file's banned-examples list.

- [ ] **Step 2: Confirm both files contain the optional-evidence schema field comment.**

Run: `grep -c "<optional body of the comment" plugins/code-review/commands/review.md plugins/code-review/commands/review-discussion.md`
Expected: each file reports exactly 1.

- [ ] **Step 3: Confirm the substantive style-rules wording is identical between the two files.**

The two files use different *heading* lines for the block (`review.md` introduces it as a numbered "Comment style rules (verbatim):" step; `review-discussion.md` introduces it as a bullet "Comment style rules (apply to every entry in `suggested_findings`):"). The substantive *body* must match word-for-word. Diff with leading-whitespace stripped to ignore the indentation difference (5 vs 7 spaces):

```bash
diff <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review.md | sed 's/^[[:space:]]*//') \
     <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review-discussion.md | sed 's/^[[:space:]]*//')
```

Expected: empty output (zero diff after leading-whitespace strip). If the diff is non-empty, edit one file to match the other so the substantive content stays aligned, then re-run.

- [ ] **Step 4: Confirm body-format prose was updated in both files.**

Run: `grep -c "If \`evidence\` is empty/null, omit this paragraph" plugins/code-review/commands/review.md plugins/code-review/commands/review-discussion.md`
Expected: each file reports exactly 1.

- [ ] **Step 5: Confirm merge step was updated in `review.md` only.**

Run: `grep -n "Concatenate non-empty evidence" plugins/code-review/commands/review.md`
Expected: exactly one hit.

Run: `grep -n "Concatenate non-empty evidence" plugins/code-review/commands/review-discussion.md`
Expected: zero hits (review-discussion has no merge step — single-agent flow).

- [ ] **Step 6: Confirm footer markers did NOT leak across files.**

Run: `grep -c "🤖 _Agent review" plugins/code-review/commands/review.md`
Expected: 2 (heredoc + body-format prose).

Run: `grep -c "👤 _Human review" plugins/code-review/commands/review-discussion.md`
Expected: 2 (heredoc + body-format bullets).

Run: `grep -c "🤖 _Agent review" plugins/code-review/commands/review-discussion.md`
Expected: 0.

Run: `grep -c "👤 _Human review" plugins/code-review/commands/review.md`
Expected: 0.

- [ ] **Step 7: Bump the plugin version.**

Use the `Edit` tool.

`old_string`:

```
  "version": "0.3.0",
```

`new_string`:

```
  "version": "0.4.0",
```

- [ ] **Step 8: Verify the version file.**

Read `plugins/code-review/.claude-plugin/plugin.json`. Confirm:
  - `"version": "0.4.0"`.
  - `"name"`, `"description"`, `"author"` fields are unchanged.

- [ ] **Step 9: Commit the version bump (and any consistency fix-ups from steps 1–6).**

```bash
git add plugins/code-review/.claude-plugin/plugin.json plugins/code-review/commands/
git commit -m "$(cat <<'EOF'
chore(code-review): bump version to 0.4.0

Optional/conversational evidence change is user-visible — every posted
comment can now skip the body when the message + code is self-explanatory,
and bodies that do appear no longer cite specific source artifacts.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If steps 1–6 surfaced no fix-ups, the commit only stages `plugin.json`. That's fine; just adjust the `git add` arguments accordingly.

- [ ] **Step 10: Recommended manual smoke test.**

These changes affect agent-prompt markdown, not executable code. There is no automated harness. Before merging, the implementer should:

1. Run `/review` against a small open PR with a known issue (or create a throwaway PR with a deliberate bug). Confirm at least one posted inline comment is just `<message>` + anchored code (no body) — proving "evidence is optional" landed end-to-end.
2. From the same run, confirm at least one posted comment that *does* have a body reads conversationally and contains zero `AGENTS.md §`, `path:Lx-Ly precedent`, or `<section-slug> rule X` framing.
3. Run `/review-discussion <pr> <question>` against an open PR with a question that surfaces at least one suggested finding. Confirm the optional pending review's inline comment shows the same shape: optional body, conversational WHY when present, no source-artifact citations.
4. Confirm the footers (`🤖 _Agent review — <sections>_` for `/review`, `👤 _Human review, helped by agent_` for `/review-discussion`) are unchanged.

If either flow regresses (e.g. agent still cites a specific section number, or omits a body where one was actually needed), capture the comment text and tighten the wording before re-shipping — don't paper over with a follow-up bullet.

Document the smoke-test result in the PR description.
