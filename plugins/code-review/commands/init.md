---
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
description: Scaffold .claude/code-review/best-practices.md for the current project
disable-model-invocation: false
---

You are scaffolding `.claude/code-review/best-practices.md` for the current project. This file is the artifact consumed by the rest of the `code-review` plugin (`/gather-insight-discussion`, `/gather-insight-pr`, `/review`, `/review-discussion`).

The arguments passed to this command are: `$ARGUMENTS`.

Your job is to walk the user through configuring sections, refs, models, and globals — then write the file. Follow the steps below in order. Do not skip steps. Do not add behaviors that aren't here.

## Quick mode

If `$ARGUMENTS` contains the literal token `--quick`:

1. Skip ALL prompts. Do not ask the user any questions in quick mode — not the existence prompt, not refs, not models, not globals, not custom sections. Nothing.
2. Behave as if any existing file is being **replaced** (overwrite at write time).
3. Use these defaults verbatim:
   - The seven default sections, each with `refs: []` and `model: inherit`, in this order: `security`, `bugs`, `performance`, `simplicity`, `testing`, `error_handling`, `conventions`.
   - `min_confidence: 80`.
   - `skip_authors: [dependabot[bot], renovate[bot], github-actions[bot]]`.
4. Render the file directly from the template (see "Render" below) and write it. Then print the summary and exit.

If `$ARGUMENTS` does NOT contain `--quick`, run the interactive flow below.

## Existence check

Run Bash:

```bash
test -f .claude/code-review/best-practices.md && echo EXISTS || echo MISSING
```

If the output is `EXISTS` and you are NOT in `--quick` mode, prompt the user EXACTLY:

```
.claude/code-review/best-practices.md already exists. (r)eplace / (m)erge / (a)bort? [a]
```

Parse the user's reply (case-insensitive, single character; empty defaults to `a`):

- `a` (or empty): exit immediately. Print `Aborted; no changes.` and stop. Make no edits.
- `r`: continue with the normal flow. At write time, overwrite the existing file.
- `m`: continue with the normal flow. At write time, **merge** with the existing file (see "Merge mode" under "Render").

If the file does not exist, proceed without prompting.

In `--quick` mode, behave as if the user picked `r` (overwrite without prompting).

Do not skip this prompt in non-quick mode if the file exists.

## Auto-discovery

Before walking sections, scan the target project and skill directories for existing artifacts that could become refs. Run these probes from the target-project root once, up front, so suggestions are ready to surface during the section walk.

### File patterns

Use `Glob` or `Bash find -maxdepth 4 . -type f` and filter. For each section below, collect matching paths:

| Section | File patterns |
|---------|---------------|
| `security` | `SECURITY.md`, anything under `docs/security/`, files whose name contains `security` (case-insensitive) |
| `conventions` | `CONTRIBUTING.md`, `STYLE.md`, `AGENTS.md`, root `CLAUDE.md` |
| `performance` | anything under `docs/performance/`, files whose name contains `perf` or `performance` (case-insensitive) |
| `simplicity` | (no file patterns; skill probe only) |
| `testing` | anything under `docs/testing/`, `TESTING.md` |
| `error_handling` | (no file patterns; skill probe only) |
| `bugs` | (no file patterns; skill probe only) |

Examples of usable Bash:

```bash
find . -maxdepth 4 -type f \( -iname 'SECURITY.md' -o -ipath '*/docs/security/*' -o -iname '*security*' \) 2>/dev/null
find . -maxdepth 4 -type f \( -iname 'CONTRIBUTING.md' -o -iname 'STYLE.md' -o -iname 'AGENTS.md' -o -iname 'CLAUDE.md' \) 2>/dev/null
find . -maxdepth 4 -type f \( -ipath '*/docs/performance/*' -o -iname '*perf*' -o -iname '*performance*' \) 2>/dev/null
find . -maxdepth 4 -type f \( -ipath '*/docs/testing/*' -o -iname 'TESTING.md' \) 2>/dev/null
```

Strip noise (e.g. `.git/`, `node_modules/`, `.venv/`, `dist/`, `build/`).

### Skill probe paths

For skills, look for `SKILL.md` files under each path below, in this order. Use the project root for project-local paths and `$HOME` for the user-global ones.

1. `.claude/skills/<name>/SKILL.md`
2. `.agents/skills/<name>/SKILL.md`
3. `~/.claude/skills/<name>/SKILL.md`
4. `~/.claude/plugins/cache/*/*/skills/<name>/SKILL.md`

Example Bash to enumerate available skills (read each `SKILL.md`'s frontmatter `name:` field — names may differ from directory names):

```bash
for d in .claude/skills .agents/skills "$HOME/.claude/skills" "$HOME"/.claude/plugins/cache/*/*/skills; do
  [ -d "$d" ] || continue
  for s in "$d"/*/SKILL.md; do
    [ -f "$s" ] || continue
    echo "$s"
  done
done
```

For each `SKILL.md` you find, extract the `name:` from its YAML frontmatter (the first `---` block) using `grep -m1 '^name:' "$s" | sed 's/^name:[[:space:]]*//'`. Match case-insensitively against the patterns below.

### Skill suggestion mapping

| Section | Skill name pattern (case-insensitive substring) |
|---------|------------------------------------------------|
| `security` | `*security*`, `owasp*` |
| `performance` | `*performance*`, `*perf*` |
| `simplicity` | `code-simplifier`, `*simplif*` |
| `testing` | `*test*`, but EXCLUDE any whose name contains `unit-test` |

Sections not listed above (`bugs`, `conventions`, `error_handling`) get no skill suggestions.

When you surface a skill suggestion to the user, format it as `skill:<name>` (the form the artifact expects in `refs`).

## Walk sections

Skip this entire section if `--quick` mode is active.

Walk the seven default sections in this exact order:

1. `security`
2. `bugs`
3. `performance`
4. `simplicity`
5. `testing`
6. `error_handling`
7. `conventions`

For each section in order:

1. Print a banner heading. Use the title-cased section name. Examples: `## Security`, `## Bugs`, `## Performance`, `## Simplicity`, `## Testing`, `## Error Handling`, `## Conventions`.
2. Surface auto-discovery suggestions on a single line below the banner if any matches were found:
   ```
   Found: SECURITY.md, docs/security/auth.md, skill:owasp-defenses
   ```
   If nothing was found for this section, omit this line (or print `Found: (none)`).
3. Ask the refs question EXACTLY:
   ```
   Refs for this section? (paths or skill:<name>, comma-separated, or empty)
   ```
   Parse the response: split on commas, strip whitespace from each item, drop empty items. Empty input becomes `[]`. Suggestions are NOT auto-applied — the user must include them in their reply if they want them.
4. Ask the model question EXACTLY:
   ```
   Model? (inherit/sonnet/opus/haiku) [inherit]
   ```
   Validate: the answer must be one of `inherit`, `sonnet`, `opus`, `haiku` (case-insensitive). Empty input → `inherit`. On invalid input, echo what was received and re-ask:
   ```
   Got "<received>", which is not one of inherit/sonnet/opus/haiku. Try again.
   Model? (inherit/sonnet/opus/haiku) [inherit]
   ```

Record `{refs, model}` for each section in your in-memory state.

## Custom sections

After the seven defaults, ask:

```
Add additional section? (name or done) [done]
```

Loop:

- Empty input or `done` (case-insensitive): exit the custom-section loop.
- Otherwise treat the input as a section name. Slugify it with these rules:
  - Lowercase the whole string.
  - Replace any whitespace runs with a single underscore (`_`).
  - Drop any character that is not `[a-z0-9_]`.
- After slugifying, verify the slug is non-empty and not already in use (defaults plus any prior custom sections). If it collides or is empty, echo what was received and re-ask:
  ```
  Slug "<slug>" is empty or already in use. Try again.
  Add additional section? (name or done) [done]
  ```
- For an accepted custom section: ask the same `Refs for this section?` and `Model?` questions as in "Walk sections" above. Record `{slug, title, refs, model}`. The display title is the original input with leading/trailing whitespace stripped — keep its capitalization for the body heading.
- Loop back to the `Add additional section?` prompt.

Do not invent additional sections beyond what the user explicitly provides.

## Globals

Skip this section if `--quick` mode is active (use the documented defaults: `min_confidence: 80`, `skip_authors: [dependabot[bot], renovate[bot], github-actions[bot]]`).

### Confidence threshold

Ask EXACTLY:

```
Confidence threshold? [80]
```

Parse as integer. Empty → `80`. Validate: integer between 0 and 100 inclusive. On invalid input, echo and re-ask:

```
Got "<received>", which is not an integer between 0 and 100. Try again.
Confidence threshold? [80]
```

Store as `min_confidence`.

### Skip PR authors

Ask EXACTLY:

```
Skip PR authors? [dependabot[bot], renovate[bot], github-actions[bot]]
```

The default list is ALWAYS present in the final config — user input is **appended**, not replacing. Parse the response: split on commas, strip whitespace, drop empties, drop entries already in the default list. Empty input means "no additions".

Final `skip_authors` list = default list + user additions, preserving order, deduplicated.

## Render

Read the template at `${CLAUDE_PLUGIN_ROOT}/templates/best-practices.template.md` (this is a Bash-resolvable path; use `Read` after expanding the variable, e.g. `bash -c 'echo $CLAUDE_PLUGIN_ROOT/templates/best-practices.template.md'`).

Build the new file as follows.

### Frontmatter

Replace the template's `sections:` map with the user-defined sections. Output order: defaults first (in canonical order, but only those the user kept — all seven are always kept since the section walk does not let the user remove defaults), then customs in the order entered. Each section emits:

```yaml
  <slug>:
    refs:
      - <ref1>
      - <ref2>
    model: <model>
```

If `refs` is empty, render `refs: []` on a single line. Quote each ref with double quotes when it contains characters that need quoting in YAML (colons, brackets, leading sigils); otherwise emit it bare. The `skill:<name>` form contains a colon, so always quote it: `"skill:<name>"`.

Set `min_confidence:` to the integer the user chose (or 80).

Set `skip_authors:` to the merged list as a YAML block sequence:

```yaml
skip_authors:
  - dependabot[bot]
  - renovate[bot]
  - github-actions[bot]
  - <user-added-1>
```

Preserve any other top-level frontmatter keys that exist in the template but are not `sections`, `min_confidence`, or `skip_authors` — pass them through verbatim. (Forward compatibility: future template revisions may add fields.)

### Body

Keep these template parts verbatim:

- The opening heading `# Code Review Best Practices`.
- The introductory paragraphs (the lines starting with `This file is consumed...` and the two bullet points after).
- The `## Rule format` block, including its fenced code example.

After `## Rule format` and its content, emit one section per user-defined section, in the same order as the frontmatter. For each:

- Heading: `## <Title>`. Titles for the seven defaults are: `Security`, `Bugs`, `Performance`, `Simplicity`, `Testing`, `Error Handling`, `Conventions`. For custom sections, use the user's original input (whitespace-trimmed) as the title.
- Body: a single placeholder comment line `<!-- Add inline rules here, or rely on refs above -->`.

Do not auto-fill any rule bullets. The placeholder stays.

### Merge mode

If the existence check resolved to `m`erge, you must preserve user-authored content from the existing file:

- Frontmatter merge: union of section keys (defaults plus any custom sections that exist in either file). For overlapping keys, prefer the values the user just chose in the walkthrough (refs/model). For sections present only in the existing file (custom sections the user didn't re-add this run), keep them with their existing refs/model. For `skip_authors`, take the union (preserve order: existing entries first, then any new additions). For `min_confidence`, prefer the new user-chosen value.
- Body merge: for each `## <Heading>` in the merged section list, look up the existing file's body for the same heading. If the existing body under that heading is non-empty AND is not just the placeholder comment, preserve that existing body verbatim under the heading. If the existing body is missing or only the placeholder, write the placeholder comment (so empty sections remain consistent). For sections that are new (no existing body), write the placeholder comment.

## Write

1. Ensure the parent directory exists:
   ```bash
   mkdir -p .claude/code-review
   ```
2. Use the `Write` tool to save the rendered content to `.claude/code-review/best-practices.md` (path relative to the current target-project root).

## Summary

After writing, print EXACTLY:

```
Created .claude/code-review/best-practices.md with N sections (M custom).
Run /gather-insight-discussion or /gather-insight-pr to populate rules.
```

Where:
- `N` is the total section count (defaults + customs).
- `M` is the count of custom sections only (0 if none).

## Hard rules

These constraints are non-negotiable. Re-read them before each step:

- **Do not invent additional sections** beyond what the user provides. The seven defaults are fixed; custom sections come only from the user's explicit input in the custom-section loop.
- **Do not auto-fill rule bodies** during init. Section bodies stay as the placeholder comment `<!-- Add inline rules here, or rely on refs above -->`. Future commands (`/gather-insight-discussion`, `/gather-insight-pr`) populate rules.
- **Do not skip the existence prompt** in non-quick mode if `.claude/code-review/best-practices.md` already exists. Always offer `(r)eplace / (m)erge / (a)bort` and respect the choice.
- **Do not silently lose user input.** When a question requires re-asking (invalid input), echo what was received first so the user sees what you parsed.
- **Do not ask any questions in `--quick` mode.** All defaults, no prompts, behave as if the user chose `r` if the file already exists.
- **Use only the declared tools:** `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`. Do not request anything else.
