# Natural comment format for `/review` and `/review-discussion`

## Problem

The current GitHub comments produced by `/review` and `/review-discussion` are unnatural. They start with a metadata header (`[<section> · confidence <N>]`) and the body — driven by the `evidence` field — tends to inflate into multi-paragraph rationale with explicit "Evidence:" footers. Reviewers don't write that way.

Concrete example seen in the wild (from `/review-discussion`):

```
[discussion · confidence 75] Rename `FcmChannels` → `FcmNotificationChannels`.

Two issues with the current name: (1) plain `Channels` is ambiguous against
Kotlin's coroutine `Channel`, and the type this object actually produces is
`NotificationChannel`; (2) keeping `Fcm` is correct here since we're explicitly
*not* centralising — `HotwordService` already owns its own foreground-service
channel inline at `core/hotword/service/HotwordService.kt#L59-L64`, so per-feature
scoping is the established pattern.

`FcmNotificationChannels` reads as "the FCM feature's notification channels" and
leaves the door open for sibling owners (e.g. a future `AlarmNotificationChannels`)
to colocate with their own feature.

Evidence: `HotwordService.kt#L59-L64` precedent (per-feature channel ownership);
code-simplifier rule "clear concept names over compound mechanical phrases."
```

A senior reviewer leaving the same comment by hand would write three sentences and stop.

## Goals

1. Comments read like a senior reviewer wrote them: imperative, short, citation inline.
2. Match the verbosity to the fix:
   - simple change → just say what to do, with a one-line "why"
   - complex change → include a concrete code suggestion or a code block showing the fix
3. Always state WHY, but keep it short unless the issue genuinely needs more.
4. Preserve a small footer marker so future runs of `/gather-insight-pr` can distinguish self-authored automated comments (low signal for rule mining) from human-driven discussion comments (high signal).
5. Use GitHub's native `suggestion` block when the fix is a literal in-place replacement of the anchored lines, so reviewers can one-click commit.

## Non-goals

- Changing the `/review` summary body (the top-of-review markdown that lists "Reviewed sections" and "Highlights"). That format is fine.
- Changing the merge logic, confidence rubric, or false-positive baseline.
- Updating `/gather-insight-pr` to act on the new footer markers. That is a follow-up task — this spec defines the markers but does not consume them.
- Migrating or rewriting comments that already exist on past PRs.

## Design

### New comment body — `/review`

```
<message>

<evidence>

---
🤖 _Agent review — <section>_
```

For findings merged across multiple sections, the footer joins all contributing section slugs with ` · `:

```
🤖 _Agent review — bugs · security_
```

The contributing-section list is the union of section slugs from the merge group, ordered by descending confidence (canonical finding's section first, then the rest in the order they were merged in). Slugs render verbatim — lowercase, underscores intact, no title-casing.

### Merge logic changes (`/review` only)

The current merge step (`review.md` step 8) keeps the highest-confidence finding as canonical and concatenates contributing evidence with ` | `. Two changes:

1. **Track the contributing-section list.** Add a `contributing_sections` field to the merged finding — an ordered list starting with the canonical finding's section, then each other contributor's section in confidence-descending order, deduplicated. Step 12's footer renders this list.

2. **Change the evidence separator from ` | ` to `\n\n`.** The new style guidance produces multi-sentence prose (sometimes with fenced code blocks). Inline ` | ` joins read awkwardly; double-newline joins read like consecutive paragraphs.

3. **One code block per merged comment.** GitHub renders only the first `suggestion` block, and stacking multiple plain code blocks from different sections is noisy. Rule: if two or more contributing findings each include a fenced code block (any language, including `suggestion`), keep the canonical finding's evidence verbatim; from non-canonical contributors, strip every fenced code block before concatenation but keep the surrounding prose. The canonical entry's code block survives intact.

The 0-findings and merge-tie-break behaviors are otherwise unchanged.

### New comment body — `/review-discussion`

```
<message>

<evidence>

---
👤 _Human review, helped by agent_
```

The footer is fixed text. There is no section variation because `/review-discussion` always runs as a single agent answering one question.

### Schema instruction rewrite

Both commands currently instruct the agent with this terse schema:

```yaml
findings:
  - path: <relative path>
    line_start: <int>
    line_end: <int>
    message: <brief, action-oriented>
    evidence: <which rule or shared context grounds this>
    confidence: <0-100>
```

(The `/review-discussion` variant uses `suggested_findings:` and identical inner fields.)

Replace the inline field comments and follow them with explicit style rules. The new schema block reads:

```yaml
findings:
  - path: <relative path>
    line_start: <int>
    line_end: <int>
    message: <imperative one-liner — what to do>
    evidence: <the body of the comment — see style rules below>
    confidence: <0-100>
```

And immediately after the schema block, append these style rules verbatim:

```
Comment style rules:

- `message` is imperative and short. For simple fixes (rename, flip a condition,
  remove an unused arg, reorder two lines), the message alone should be enough;
  the body just adds a one-line why.

- `evidence` is the body of the comment. Lead with WHY in 1–3 short sentences,
  grounded inline in the rule or code reference (e.g. "matches `path:Lx-Ly`
  precedent", "violates the `<section>` rule X"). For complex fixes, include
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

Both commands receive the same style rules block.

### Posting template changes

`review.md` step 12, currently:

```bash
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

becomes:

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

`<sections>` is the merged finding's `contributing_sections` list (defined in the merge-logic subsection above), joined with ` · `.

`review-discussion.md` step 4 of the posting flow becomes:

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

The accompanying body-format prose in both commands is updated to match (drop the bracket-header description, add the footer description, drop the `<N>` substitution).

### Worked examples

**Simple rename — the screenshot example, rewritten:**

```
Rename `FcmChannels` → `FcmNotificationChannels`.

`Channels` collides with Kotlin's coroutine `Channel`, and this object produces `NotificationChannel`. The `Fcm` prefix scopes to the feature, matching `HotwordService.kt#L59-L64`'s per-feature ownership.

​```suggestion
object FcmNotificationChannels {
​```

---
👤 _Human review, helped by agent_
```

(Anchored on the single `object FcmChannels {` line — one-click commit.)

**Cross-section merge from `/review`:**

```
Add a null-check on `session` before dereferencing on line 47.

`session` can be null right after logout (see `auth.kt:L120` early-return). Without the guard the next render crashes; the leaked stack trace also exposes the JWT prefix.

​```kotlin
val s = session ?: return
s.userId
​```

---
🤖 _Agent review — bugs · security_
```

(Plain `kotlin` block, not `suggestion`, because the fix introduces a new local and the anchor is on the dereference line, not on the two replacement lines.)

## Backward compatibility

- `/gather-insight-pr` reads PR comment bodies via the GitHub API. It does not parse the bracket header today, so dropping the header doesn't break extraction.
- Existing comments on past PRs are untouched. Mining old comments still works (the extractor sees the old `[section · confidence N]` prefix as part of the body — extra noise but not a parse failure).
- The `/review` summary body, the merge logic, the confidence filter, and the post-eligibility race guard are unchanged.

## Risks

1. **Section attribution moves from the top to the footer.** Reviewers skimming a long PR may miss it. Mitigation: italics + horizontal rule make it visually distinct, and most users skim by file/line first, section second.
2. **Confidence is no longer surfaced to the human reader.** The numeric value already had narrow signal once a comment passed `min_confidence` (default 80). Mitigation: confidence is still tracked in the artifact and used for merge tie-breaking; it just no longer pollutes the rendered comment.
3. **Agents may emit malformed `suggestion` blocks** (wrong indentation, mismatched line count). Mitigation: the schema instruction's three preconditions tell the agent to fall back to a plain block when any condition fails. We accept that some bad suggestions will still ship; reviewers can decline them in one click.
4. **Merge-footer cardinality.** A finding merged across all seven default sections would render `🤖 _Agent review — security · bugs · performance · simplicity · testing · error_handling · conventions_`. Acceptable: this only happens when seven sections all flag the exact same line span, which is rare and itself a useful signal.

## Out of scope (explicit follow-ups)

- Update `/gather-insight-pr` to detect the `🤖 Agent review` and `👤 Human review` footers and use them as filtering or prioritization signals during rule mining. Tracked separately.
