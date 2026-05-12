# Natural Comment Format Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the bracket-header comment format produced by `/review` and `/review-discussion` with a senior-reviewer-style body and a small footer marker, and teach the agent when to use GitHub's native `suggestion` block versus a plain code block.

**Architecture:** Edit-only change against two slash-command markdown files (`plugins/code-review/commands/review.md`, `plugins/code-review/commands/review-discussion.md`). No new files, no test framework — these markdown files are LLM agent prompts. Verification uses `grep` for stale references, file re-reads for internal consistency, and (recommended) a manual smoke test on a real PR.

**Tech Stack:** Markdown command files; `gh-pr-review` CLI extension (unchanged); GitHub PR review API renders `suggestion` code blocks natively.

**Spec:** `docs/superpowers/specs/2026-05-06-natural-comment-format-design.md`

---

## File Structure

Files modified by this plan:

- `plugins/code-review/commands/review.md` — schema instruction (~line 300), merge section (~line 316), posting template (~line 371), body-format prose (~line 386).
- `plugins/code-review/commands/review-discussion.md` — schema instruction for `suggested_findings` (~line 320), posting template (~line 385), body-format prose (~line 400).

No new files. The agent-prompt markdown files stay self-contained — the comment-style rules block is inlined verbatim in both commands rather than extracted to a shared partial (no template system exists, and inlining keeps each command independently readable).

A note about line numbers: edits in earlier tasks shift the line numbers cited in later tasks. Use the anchor strings (e.g. `## Merge`, the heredoc literal) shown in each task's Edit step to locate the target — do not rely on line numbers across tasks.

---

### Task 1: Update the schema and add comment-style rules in `review.md`

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the `8. **Output schema instruction:**` block under the parallel-agent prompt section.

- [ ] **Step 1: Open the file and locate the target.**

Run: `grep -n "Output schema instruction" plugins/code-review/commands/review.md`
Expected: one hit, around line 300.

- [ ] **Step 2: Replace the schema block and append the style rules.**

Use the `Edit` tool. Match the existing block exactly (including indentation — these are inside a numbered-list step, indented 5 spaces).

`old_string`:

```
  8. **Output schema instruction:**

     ```yaml
     findings:
       - path: <relative path>
         line_start: <int>
         line_end: <int>
         message: <brief, action-oriented>
         evidence: <which rule or shared context grounds this>
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
         evidence: <the body of the comment — see style rules below>
         confidence: <0-100>
     ```

     Followed by: `If no findings, return findings: [].`

  9. **Comment style rules (verbatim):**

     ```
     - `message` is imperative and short. For simple fixes (rename, flip a condition,
       remove an unused arg, reorder two lines), the message alone should be enough;
       the body just adds a one-line why.

     - `evidence` is the body of the comment. Lead with WHY in 1–3 short sentences,
       grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly`
       precedent", "violates the <section> rule X"). For complex fixes, include
       a fenced code block showing the concrete change.

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
```

- [ ] **Step 3: Renumber any following steps that collide with the new step 9.**

The original list went `1, 2, 3, 4, 5, 6, 7, 8` and ended at step 8 ("Output schema instruction"). The Step 2 edit above adds a new step `9.` ("Comment style rules"). Re-read the surrounding context to confirm there is no pre-existing `9.` in the same list:

Run: `grep -n "^  9\." plugins/code-review/commands/review.md`
Expected: exactly one hit (the new step 9 just inserted). If more than one hit, renumber the lower one and any sibling steps.

- [ ] **Step 4: Verify the file still parses as expected by re-reading the section.**

Read `plugins/code-review/commands/review.md` lines 295–355. Confirm:
  - The new step 9 sits between the schema block and the "Once all parallel `Agent` calls complete..." paragraph.
  - The next `## ` heading after the inserted block is still `## Merge` (Task 2 will edit it).
  - The schema's `message` and `evidence` field comments now read `imperative one-liner — what to do` and `the body of the comment — see style rules below`.

- [ ] **Step 5: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): tighten /review schema + add comment-style rules

Rewrite the schema field comments and append a verbatim "Comment style rules"
block telling the agent how to write `message`/`evidence` and when to use
GitHub suggestion blocks vs plain code blocks.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Update merge logic in `review.md` (contributing_sections, separator, code-block dedup)

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the `## Merge` section and the preceding "Once all parallel `Agent` calls complete..." paragraph that mentions section tagging.

- [ ] **Step 1: Locate the section-tagging paragraph.**

Run: `grep -n "Tag each finding with the originating section slug" plugins/code-review/commands/review.md`
Expected: exactly one hit (around line 314 before edits, but Task 1's edits shifted it down).

- [ ] **Step 2: Update the section-tagging paragraph.**

Use `Edit`. The current sentence references "step 12 can render the section in the comment header" — this is now stale.

`old_string`:

```
Once all parallel `Agent` calls complete, collect every finding into a flat list. **Tag each finding with the originating section slug** (so step 12 can render the section in the comment header). Findings outside that schema are ignored.
```

`new_string`:

```
Once all parallel `Agent` calls complete, collect every finding into a flat list. **Tag each finding with the originating section slug** (the merge step uses this to build the merged finding's `contributing_sections` list, which the posting template renders in the comment footer). Findings outside that schema are ignored.
```

- [ ] **Step 3: Locate the Merge section.**

Run: `grep -n "^## Merge" plugins/code-review/commands/review.md`
Expected: one hit. Read the next ~12 lines so you know exactly what you're replacing.

- [ ] **Step 4: Replace the Merge section body.**

Use `Edit`.

`old_string`:

```
## Merge

Walk the flat finding list and group findings whose `(path, line_start, line_end)` overlap within ±2 lines on either edge. For each group:

- Keep the **highest-confidence** finding as the canonical one.
- **Preserve evidence from both** in the canonical finding's `evidence` field — concatenate, separated by ` | ` so the eventual inline comment shows reasoning from every contributing section.

This collapses cross-section duplicates (e.g. `bugs` and `security` both flagging a missing null check on the same line). Do not lose evidence — keep all of it on the survivor.
```

`new_string`:

```
## Merge

Walk the flat finding list and group findings whose `(path, line_start, line_end)` overlap within ±2 lines on either edge. For each group:

- Keep the **highest-confidence** finding as the canonical one.
- **Build a `contributing_sections` list** on the canonical finding: an ordered, deduplicated list starting with the canonical finding's section slug, followed by each other contributor's section slug in confidence-descending order. The posting template renders this list in the comment footer (e.g. `bugs · security`).
- **Concatenate evidence with `\n\n`** (a blank line between paragraphs), in confidence-descending order. Inline ` | ` joins read awkwardly when evidence contains multi-sentence prose or fenced code blocks; double-newline joins read like consecutive paragraphs.
- **One code block per merged comment.** GitHub renders only the first `suggestion` block, and stacking plain code blocks from different sections is noisy. Rule: if two or more contributing findings each include a fenced code block (any language, including `suggestion`), keep the canonical finding's evidence verbatim; from non-canonical contributors, **strip every fenced code block before concatenation** (delete the entire ` ```...``` ` span) but keep the surrounding prose. The canonical entry's code block survives intact.

This collapses cross-section duplicates (e.g. `bugs` and `security` both flagging a missing null check on the same line). Do not lose prose evidence — every contributor's reasoning still reaches the merged comment.
```

- [ ] **Step 5: Verify by re-reading.**

Read `plugins/code-review/commands/review.md` from the new step 9 (Comment style rules) through the end of the `## Merge` section. Confirm:
  - The section-tagging paragraph references `contributing_sections` and "comment footer".
  - The Merge bullets list `contributing_sections`, `\n\n` separator, and the code-block dedup rule.
  - No leftover ` | ` separator references in this section.

Run: `grep -n "separated by \` | \`" plugins/code-review/commands/review.md`
Expected: zero hits.

- [ ] **Step 6: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): track contributing_sections + tighten merge for natural body

Merge step now builds an ordered contributing_sections list (used by the
footer), joins evidence with \n\n instead of ` | `, and keeps only the
canonical finding's fenced code block when multiple contributors have one.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Update the posting template and body-format prose in `review.md`

**Files:**

- Modify: `plugins/code-review/commands/review.md` — the `gh pr-review review --add-comment` heredoc and the "Body format (must follow exactly):" prose that documents it.

- [ ] **Step 1: Locate the heredoc.**

Run: `grep -n "\[<section> · confidence <N>\]" plugins/code-review/commands/review.md`
Expected: two hits — one inside the heredoc, one in the "Body format" prose immediately after.

- [ ] **Step 2: Replace the heredoc.**

Use `Edit`.

`old_string`:

```
gh pr-review review --add-comment \
  --review-id <review-id> \
  --pr <pr-number> -R <owner>/<repo> \
  --path <path> \
  --line <line_end> \
  --start-line <line_start> \
  --body "$(cat <<EOF
[<section> · confidence <N>] <message>

<evidence>
EOF
)"
```

`new_string`:

```
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

- [ ] **Step 3: Replace the body-format prose.**

Use `Edit`.

`old_string`:

```
**Body format (must follow exactly):**

- Header line: `[<section> · confidence <N>] <one-line action>` where `<section>` is the originating section slug, `<N>` is the merged confidence (after step 8), and `<one-line action>` is the finding's `message`.
- Blank line.
- `<evidence>` paragraph (the merged evidence string from step 8, including any `|`-joined cross-section evidence).
```

`new_string`:

```
**Body format (must follow exactly):**

- `<message>` line: the finding's `message` field — an imperative one-liner.
- Blank line.
- `<evidence>` paragraph(s): the merged evidence string from step 8 (paragraphs joined by blank lines; may include a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise).
- Blank line, then the literal separator `---` on its own line, then a blank line.
- Footer line: `🤖 _Agent review — <sections>_` where `<sections>` is the merged finding's `contributing_sections` list joined with ` · ` (space–middle-dot–space). Slugs render verbatim — lowercase, underscores intact, no title-casing. For a single-section finding this is just the one slug (e.g. `🤖 _Agent review — bugs_`).
```

- [ ] **Step 4: Sanity check — no stale bracket-header references in this file.**

Run: `grep -n "\[<section>\|\[<section> · confidence\|· confidence <N>" plugins/code-review/commands/review.md`
Expected: zero hits.

Run: `grep -n "🤖 _Agent review" plugins/code-review/commands/review.md`
Expected: two hits (one in the heredoc, one in the body-format prose).

- [ ] **Step 5: Verify by re-reading.**

Read `plugins/code-review/commands/review.md` from the `## Pending review` heading through the end of the body-format prose. Confirm:
  - The heredoc renders message, evidence, separator, footer in that order.
  - The body-format prose mentions `contributing_sections`, the `---` separator, and the `🤖 _Agent review — <sections>_` footer line.
  - No leftover `[<section> · confidence <N>]` references anywhere in the file.

- [ ] **Step 6: Commit.**

```bash
git add plugins/code-review/commands/review.md
git commit -m "$(cat <<'EOF'
feat(code-review): drop bracket header, add agent-review footer in /review

The per-finding inline comment body now renders message, evidence, then a
'🤖 _Agent review — <contributing_sections>_' footer separated by ---.
Documents the new shape in the body-format prose so future edits stay aligned.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Update the schema and add comment-style rules in `review-discussion.md`

**Files:**

- Modify: `plugins/code-review/commands/review-discussion.md` — the `suggested_findings` schema block in the answering-agent prompt section.

- [ ] **Step 1: Locate the schema block.**

Run: `grep -n "suggested_findings:" plugins/code-review/commands/review-discussion.md`
Expected: at least one hit inside a YAML schema block in the "Answering instructions" step.

- [ ] **Step 2: Replace the schema block and append style rules.**

Use `Edit`. The schema sits inside a nested-list step indented 7 spaces (each item is `- ` under a numbered step). Match the existing indentation exactly.

`old_string`:

```
     - If the answer surfaces concrete actionable issues, append a `### Suggested findings` block at the very bottom in this YAML schema:

       ```yaml
       suggested_findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <brief, action-oriented>
           evidence: <which loaded resource grounds this>
           confidence: <0-100>
       ```

       Each finding's `(line_start, line_end)` MUST be inside the in-scope diff loaded in step 5 — never anchor a suggestion to a line outside what the answering agent saw. If no actionable issues, omit the block entirely (do **not** emit `suggested_findings: []`).
```

`new_string`:

```
     - If the answer surfaces concrete actionable issues, append a `### Suggested findings` block at the very bottom in this YAML schema:

       ```yaml
       suggested_findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <imperative one-liner — what to do>
           evidence: <the body of the comment — see style rules below>
           confidence: <0-100>
       ```

       Each finding's `(line_start, line_end)` MUST be inside the in-scope diff loaded in step 5 — never anchor a suggestion to a line outside what the answering agent saw. If no actionable issues, omit the block entirely (do **not** emit `suggested_findings: []`).

     - Comment style rules (apply to every entry in `suggested_findings`):

       ```
       - `message` is imperative and short. For simple fixes (rename, flip a condition,
         remove an unused arg, reorder two lines), the message alone should be enough;
         the body just adds a one-line why.

       - `evidence` is the body of the comment. Lead with WHY in 1–3 short sentences,
         grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly`
         precedent", "violates the <section> rule X"). For complex fixes, include
         a fenced code block showing the concrete change.

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
```

- [ ] **Step 3: Verify by re-reading.**

Read the surrounding section (the "Answering instructions" step) end-to-end. Confirm:
  - The new bullet starts with "Comment style rules (apply to every entry in `suggested_findings`):" and is followed by the verbatim block.
  - The bullet sits between the YAML schema bullet and the next list item (the "When the answering agent returns..." paragraph that closes the step).
  - Indentation aligns with sibling bullets.

Run: `grep -c "Comment style rules" plugins/code-review/commands/review-discussion.md`
Expected: 1.

- [ ] **Step 4: Commit.**

```bash
git add plugins/code-review/commands/review-discussion.md
git commit -m "$(cat <<'EOF'
feat(code-review): tighten /review-discussion schema + add comment-style rules

Mirror the /review change in /review-discussion: refresh the suggested_findings
field comments and append the verbatim Comment style rules block (message
discipline, suggestion-vs-plain-block preconditions).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Update the posting template and body-format prose in `review-discussion.md`

**Files:**

- Modify: `plugins/code-review/commands/review-discussion.md` — the `gh pr-review review --add-comment` heredoc inside the posting flow, and the bullets that document the body format.

- [ ] **Step 1: Locate the heredoc.**

Run: `grep -n "\[discussion · confidence <N>\]" plugins/code-review/commands/review-discussion.md`
Expected: two hits — one inside the heredoc, one in the bullets that follow.

- [ ] **Step 2: Replace the heredoc.**

Use `Edit`. The heredoc is inside a numbered-list step indented 3 spaces.

`old_string`:

```
   ```bash
   gh pr-review review --add-comment \
     --review-id <review-id> \
     --pr <pr-number> -R <owner>/<repo> \
     --path <path> \
     --line <line_end> \
     --start-line <line_start> \
     --body "$(cat <<EOF
   [discussion · confidence <N>] <message>

   <evidence>
   EOF
   )"
   ```
```

`new_string`:

```
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
   👤 _Human review, helped by agent_
   EOF
   )"
   ```
```

- [ ] **Step 3: Replace the body-format bullets.**

Use `Edit`.

`old_string`:

```
   - Header line is literally `[discussion · confidence <N>] <message>` (the section is always `discussion` here — there are no parallel section agents).
   - `<N>` is the finding's `confidence`.
   - `<message>` is the finding's `message`.
   - `<evidence>` is the finding's `evidence`.
```

`new_string`:

```
   - `<message>` line: the finding's `message` field — an imperative one-liner.
   - Blank line, then `<evidence>` paragraph(s): the finding's `evidence` field (may contain a fenced code block — `suggestion` if it cleanly replaces the anchored lines, plain ` ```<lang> ` otherwise).
   - Blank line, then the literal separator `---` on its own line, then a blank line.
   - Footer line: literally `👤 _Human review, helped by agent_`. This is fixed text — `/review-discussion` always runs as a single agent answering one question, so there are no per-section variations.
```

- [ ] **Step 4: Sanity check — no stale bracket-header references in this file.**

Run: `grep -n "\[discussion · confidence\|\[discussion ·" plugins/code-review/commands/review-discussion.md`
Expected: zero hits.

Run: `grep -n "👤 _Human review" plugins/code-review/commands/review-discussion.md`
Expected: two hits (heredoc and body-format bullets).

- [ ] **Step 5: Verify by re-reading.**

Read the surrounding posting-flow section end-to-end. Confirm:
  - The heredoc renders message, evidence, separator, footer in that order.
  - The body-format bullets describe the new shape.
  - The next bullets ("If `line_end == line_start`, omit `--start-line`...", "Anchors must be inside the diff...") are still present and correctly attached to the step.

- [ ] **Step 6: Commit.**

```bash
git add plugins/code-review/commands/review-discussion.md
git commit -m "$(cat <<'EOF'
feat(code-review): drop bracket header, add human-review footer in /review-discussion

The per-finding inline comment body now renders message, evidence, then the
fixed '👤 _Human review, helped by agent_' footer separated by ---. Documents
the new shape in the body-format bullets so future edits stay aligned.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Final cross-file consistency pass

**Files:**

- Read-only verification across `plugins/code-review/`. May produce a final fix-up commit if drift is found.

- [ ] **Step 1: Confirm no leftover bracket-header references anywhere in the plugin.**

Run: `grep -rn "\[<section> · confidence\|\[discussion · confidence\|\[<section>\|· confidence <N>" plugins/code-review/`
Expected: zero hits.

If any hits exist, edit the file to remove or update them, then loop back to step 1 until clean.

- [ ] **Step 2: Confirm both files contain the comment-style rules block.**

Run: `grep -c "Comment style rules" plugins/code-review/commands/review.md plugins/code-review/commands/review-discussion.md`
Expected: each file reports exactly 1.

- [ ] **Step 3: Confirm the style-rules wording is identical between the two files.**

The two files use different *heading* lines for the block (review.md introduces it as a numbered "Comment style rules (verbatim):" step; review-discussion.md introduces it as a bullet "Comment style rules (apply to every entry in `suggested_findings`):"). The substantive *body* of the block must match word-for-word. Anchor the diff on the first substantive line and the last:

```bash
diff <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review.md | sed 's/^[[:space:]]*//') \
     <(sed -n '/- `message` is imperative/,/One suggestion block per comment/p' \
        plugins/code-review/commands/review-discussion.md | sed 's/^[[:space:]]*//')
```

Expected: empty output (zero diff after leading-whitespace strip). If the diff is non-empty, edit one file to match the other so the substantive content stays aligned.

- [ ] **Step 4: Confirm both footer markers exist in the right files.**

Run: `grep -c "🤖 _Agent review" plugins/code-review/commands/review.md`
Expected: 2 (heredoc + body-format prose).

Run: `grep -c "👤 _Human review" plugins/code-review/commands/review-discussion.md`
Expected: 2 (heredoc + body-format bullets).

Run: `grep -c "🤖 _Agent review" plugins/code-review/commands/review-discussion.md`
Expected: 0 (the agent footer must not leak into the discussion command).

Run: `grep -c "👤 _Human review" plugins/code-review/commands/review.md`
Expected: 0 (the human footer must not leak into the review command).

- [ ] **Step 5: Confirm `contributing_sections` is referenced where expected in `review.md`.**

Run: `grep -n "contributing_sections" plugins/code-review/commands/review.md`
Expected: at least three hits — section-tagging paragraph, Merge bullets, body-format prose.

Run: `grep -n "contributing_sections" plugins/code-review/commands/review-discussion.md`
Expected: zero hits (this field exists only in the multi-section `/review` flow).

- [ ] **Step 6: Recommended manual smoke test.**

These changes affect agent-prompt markdown, not executable code. There is no automated harness. Before merging, the implementer should:

1. Run `/review` against a small open PR with a known issue (or create a throwaway PR with a deliberate bug). Confirm the posted inline comments use the new format: no bracket header, body matches the message/evidence shape, footer reads `🤖 _Agent review — <section>_`.
2. Run `/review-discussion <pr> <question>` against the same or a different open PR with a question that surfaces at least one suggested finding. Confirm the optional pending review's inline comment uses the new format with the `👤 _Human review, helped by agent_` footer.
3. If either flow produces an unwanted artifact (e.g. agent still emits an "Evidence:" footer paragraph, or a `suggestion` block that fails GitHub's apply check), capture the comment body and open a follow-up to refine the style-rules wording. Don't bypass it inline.

Document the smoke-test result in the PR description.

- [ ] **Step 7: Commit (only if Steps 1–5 produced fix-ups).**

If any of the consistency checks above triggered an edit, commit the fix-ups now:

```bash
git add plugins/code-review/commands/
git commit -m "$(cat <<'EOF'
fix(code-review): align comment-format edits across review/review-discussion

Cross-file consistency pass: ensures the style-rules block, footer markers,
and contributing_sections references are coherent between the two commands.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If Steps 1–5 all passed without edits, skip the commit.
