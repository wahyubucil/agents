# Code Review Plugin — Design

**Date:** 2026-05-05
**Status:** Draft, pending implementation plan
**Owner:** wahyubucil

## Overview

A personal Claude Code plugin that captures project-specific code-review knowledge in a single committed artifact, then applies it to pull requests via parallel section-aware agents. The plugin lives inside the existing `wahyubucil/agents` repo as a sibling to the `skills/` directory.

Five slash commands form a closed loop:

1. **`/init`** — scaffold the per-project best-practices file
2. **`/gather-insight-discussion`** — distill rules from a conversation, append to the file
3. **`/gather-insight-pr <pr>`** — mine human reviewer feedback from a past PR, append new rules
4. **`/review [pr]`** — run the full review using parallel per-section agents, post a pending GitHub review with inline comments
5. **`/review-discussion [pr] <question>`** — focused Q&A over a PR with lazy loading of diff/sections/refs and access to ad-hoc skills, MCP tools, and web; optionally roll the answer into a pending review

Reviews are powered by the [`agynio/gh-pr-review`](https://github.com/agynio/gh-pr-review) `gh` CLI extension (not GitHub MCP), which provides structured review viewing and pending-review GraphQL helpers.

## Goals

- Make project-specific review knowledge a versioned, committable artifact rather than a scattered set of CLAUDE.md prose.
- Let each review section run with its own model so security/correctness can use opus while style can use sonnet/haiku.
- Re-use what the project already has (`SECURITY.md`, `AGENTS.md`, custom skills) by reference rather than copying rules.
- Convert human PR reviews into permanent rules instead of letting that knowledge evaporate.
- Pending reviews by default — never auto-submit a review on the user's behalf.

## Non-goals

- Public marketplace distribution. Personal plugin in the user's `agents/` repo for now.
- Bundling external skills. Skills referenced by the artifact (`code-simplifier`, etc.) stay where they live; the plugin only references them.
- Build/typecheck/lint duplication. CI handles those; review agents are explicitly told not to.
- Replacing CLAUDE.md. The best-practices artifact lives alongside CLAUDE.md and serves a narrower purpose.

## Plugin layout

```
agents/
  plugins/
    code-review/
      .claude-plugin/
        plugin.json
      commands/
        init.md
        gather-insight-discussion.md
        gather-insight-pr.md
        review.md
        review-discussion.md
      templates/
        best-practices.template.md
      README.md
```

`agents/` becomes a local marketplace; the plugin is installed via `/plugin install ./plugins/code-review` from a local marketplace registration. No `skills/` directory inside the plugin initially — internal helpers can be added later if logic gets reused across commands.

## The artifact: `best-practices.md`

### Location

`.claude/code-review/best-practices.md` in each target project.

- Committed to git so it is a team-readable spec, not a personal lens.
- Lives under `.claude/` so it does not pollute the project's `docs/`.
- The `code-review/` directory leaves room for future siblings (templates, prompts) without renaming.

### Shape

Single file with YAML frontmatter and human-readable section bodies. Frontmatter holds machine-readable config (per-section model + refs); the body holds prose rules.

```markdown
---
sections:
  security:
    refs: ["SECURITY.md", "skill:owasp-defenses"]
    model: opus
  bugs:
    refs: []
    model: opus
  performance:
    refs: ["docs/perf.md"]
    model: sonnet
  simplicity:
    refs: ["skill:code-simplifier"]
    model: sonnet
  testing:
    refs: []
    model: inherit
  error_handling:
    refs: []
    model: inherit
  conventions:
    refs: ["AGENTS.md", "CLAUDE.md"]
    model: sonnet

min_confidence: 80
skip_authors:
  - dependabot[bot]
  - renovate[bot]
  - github-actions[bot]
---

## Security
- Always validate input at trust boundaries
  - **Why:** auth/parsing bugs at the boundary cascade everywhere
  - **Example:** `parseInt(input, 10)` not `parseInt(input)`

## Bugs
<!-- Add inline rules here, or rely on refs above -->

## Performance
- Avoid N+1 queries in loops over fetched collections

...
```

### Frontmatter schema

| Field | Type | Description |
|-------|------|-------------|
| `sections` | map | Section name → `{ refs, model }`. The frontmatter key is the slug (lowercase, underscores). The matching body heading uses Title Case (`security` → `## Security`, `error_handling` → `## Error Handling`). The plugin slugifies headings to match keys. |
| `sections.<x>.refs` | string[] | Resolvable references. Supported forms: relative file path (`docs/x.md`), absolute path, `skill:<skill-name>` (resolved via Claude Code skill discovery). |
| `sections.<x>.model` | string | One of `inherit`, `sonnet`, `opus`, `haiku`. Default `inherit`. |
| `min_confidence` | int | Default 80. Findings below this are dropped. |
| `skip_authors` | string[] | PR authors whose PRs are skipped wholesale. Default `[dependabot[bot], renovate[bot], github-actions[bot]]`. |
| `skip_review_authors` | string[] | (Optional) Reviewers whose comments `/gather-insight-pr` ignores in addition to the built-in bot list (coderabbit, sourcery, codiumai, codeball, codeguru). Self is **not** dropped. |

### Rule format

Each rule is a top-level bullet under a section heading. Optional `**Why:**` and `**Example:**` sub-bullets. Both `gather-insight-*` commands write rules in this shape.

```markdown
- **Always check for null** before accessing nested properties on API responses
  - **Why:** API returns sparse objects when fields are unauthorized
  - **Example:** `user?.profile?.name ?? 'Anonymous'`
```

## Plugin-internal behaviors (not user-editable)

These are baked into the command prompts. Users do not maintain them.

### Eligibility check (Haiku)

Run before any work. Skip the PR with a clear printed reason if any of:

- closed, merged, or draft
- author in `skip_authors`
- already reviewed at HEAD by self (delta logic below: skip only if no new commits since last review)

A second eligibility check runs after scoring (cheap Haiku call) — guards against the PR closing mid-review.

### False-positive baseline filter

Passed verbatim to every section agent:

- Pre-existing issues not introduced by the PR
- Looks like a bug but is not
- Pedantic nitpicks a senior engineer would not raise
- Linter / typechecker / compiler territory (CI handles it)
- General quality issues (test coverage, doc completeness) unless a rule explicitly demands them
- Issues silenced via `lint-ignore` / `eslint-disable` comments
- Likely intentional changes
- Issues on lines the user did not modify

### Confidence rubric

Section agents return a confidence integer per finding using this rubric:

- **0**: false positive, fails light scrutiny, or is pre-existing
- **25**: might be real, agent could not verify
- **50**: real but minor or rare in practice
- **75**: real, important, very likely to bite, or directly cited by a rule
- **100**: real, will happen frequently, evidence directly confirms

Findings below `min_confidence` (default 80) are dropped before posting.

### Citation format (summary body only)

Inline comments anchor automatically via `gh pr-review`. The summary body uses GitHub's permalink format with full SHA and line range plus ≥1 line of context:

```
https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>
```

### Build/typecheck/lint disclaimer

Section agents are told explicitly: do not run build, typecheck, or lint. CI handles those.

## Shared review context

Fetched once at the start of `/review`, shared to all section agents:

1. **Diff** — `gh pr diff <pr>` (or scoped to the delta when re-reviewing).
2. **Git blame on changed lines** — historical authorship for the modified hunks.
3. **In-code comments in modified files** — TODO/FIXME/NOTE/`@see` annotations that may guide what the diff should/should not do.
4. **Past PR comments on touched files** — last ~5 merged PRs touching the same paths and their review threads.

These are *context*, not sections. Any section agent may use them to ground a finding.

## Commands

### `/init`

**Purpose:** scaffold `.claude/code-review/best-practices.md` in the target project.

**Flow:**

1. **Existence check** — if the file already exists, prompt: `(r)eplace / (m)erge / (a)bort`. Default abort.
2. **Auto-discover existing docs** — probe the target repo and Claude skill directories, surface suggestions per section:
   - `Security` ← `SECURITY.md`, `docs/security/**`, `*security*`
   - `Conventions` ← `CONTRIBUTING.md`, `STYLE.md`, `AGENTS.md`, root `CLAUDE.md`
   - `Performance` ← `docs/performance/**`
   - All sections ← installed skills in `~/.claude/skills/`, `.claude/skills/`, `.agents/skills/`. Match by skill name (e.g. skill `code-simplifier` → `Simplicity`).
3. **Walk default sections** — Security, Bugs, Performance, Simplicity, Testing, Error Handling, Conventions. For each:
   - Show discovered suggestions
   - Ask: `Refs? (paths/skills, comma-sep, or empty)`
   - Ask: `Model? (inherit/sonnet/opus/haiku, default inherit)`
4. **Custom sections** — `Add additional section? (name or done)`. Loop.
5. **Global config** — confidence threshold (default 80), skip-authors list (default bots).
6. **Write file** — frontmatter + headings. Empty-body sections get an `<!-- Add inline rules here, or rely on refs above -->` placeholder.
7. **`--quick` flag** — skip all prompts; create the default 7 sections, `inherit` model, no refs, threshold 80, default skip-authors. For users who plan to fill via `/gather-insight-*`.

### `/gather-insight-discussion [topic?]`

**Purpose:** turn a conversation into a single distilled rule appended to the artifact.

**Flow:**

1. **Load file** — fail if not initialized; suggest `/init`.
2. **Topic from `$ARGUMENTS`** — if provided, that is the seed; otherwise prompt for one.
3. **Dialogue** — clarifying questions until the rule is one clear line plus optional `Why:` / `Example:`.
4. **Section assignment** — propose a target section based on rule content; user can override or create a new section inline.
5. **Conflict check** — semantic similarity scan against existing rules in that section. On overlap: `(r)eplace / (m)erge / (k)eep both / (s)kip`.
6. **Confirm** — show the diff (just the new lines) under the target heading. `(y)es / (e)dit / (n)o`.
7. **Append + loop** — write to file, then ask "Another? (y/n)". Default `n`.

### `/gather-insight-pr <pr>`

**Purpose:** mine human reviewer feedback from a single PR and append generalizable rules that are not already captured.

**Flow:**

1. **Resolve PR** — accepts URL, number, or fails with usage.
2. **Fetch review data** — single call: `gh pr-review review view <pr> --include-comment-node-id`. Pulls top-level review summaries and all inline thread comments in one structured tree.
3. **Filter authors:**
   - Drop bots (`[bot]` suffix or in built-in list: coderabbit, sourcery, codiumai, codeball, codeguru, github-actions, dependabot, renovate, plus `skip_review_authors` from frontmatter).
   - **Keep self.** The user often reviews other people's PRs; those comments are first-class rule sources.
4. **Filter substantive** — drop comments shorter than 20 chars, drop bare approvals/LGTM (case-insensitive match against a small list: `lgtm`, `looks good`, `approved`, `nit`, `+1`), drop emoji-only (no alphabetic characters). Everything else proceeds to extraction; the rule extractor itself decides if the comment is generalizable via the `generalizability` score in step 5.
5. **Extract rules (parallel Sonnet agents)** — one per surviving comment. Each returns:
   ```yaml
   rule: "Always check for null before accessing nested properties"
   why: "API returns sparse objects when fields are unauthorized"
   evidence: "Reviewer @alice flagged this on src/users.ts L45 in PR #123"
   suggested_section: "bugs"
   generalizability: 0-100
   ```
   Drop low-generalizability rules (project-specific renames, single-occurrence nits).
6. **Dedup against existing** — semantic match against the artifact. Bucket as `new` / `duplicate` / `refines-existing`.
7. **Present batch** — print counts ("Mined 12 comments from 3 reviewers; 5 new, 2 refine, 5 duplicates"). Prompt: `(a)ccept all / (s)elect / (n)one`. Default `(s)elect` walks one at a time.
8. **Append + summary** — write to file, print "Added 4 rules across security/bugs sections from PR #123".

### `/review [pr]`

**Purpose:** run the review against the artifact's rules and post a pending GitHub review with inline comments.

**Invocation:** all three forms supported — current branch (auto-detect via `gh pr view --json number`), PR number, or PR URL.

**Flow:**

1. **Resolve PR.** Bail with clear error if no PR is associated with current branch.
2. **Pre-flight eligibility (Haiku)** — closed/merged/draft/skip-author/already-reviewed-at-HEAD. Bail with reason.
3. **Delta computation (Q7c-ii)**:
   - `gh pr-review review view <pr> --reviewer <self>` to find the most recent prior review and its commit SHA (`commit` field on the review node).
   - If no prior review: full diff is in scope.
   - If prior review at SHA X and HEAD is at SHA Y: `git diff X..Y -- <changed-files>` is in scope. If empty, bail with "already reviewed at HEAD".
4. **Load artifact** — read `.claude/code-review/best-practices.md`. Warn (not fail) if missing — review will use a degraded mode with only the false-positive baseline and confidence rubric, no project rules.
5. **Resolve refs** — for each section's `refs`, load file content or skill content into that section's agent context. Skill resolution order: `.claude/skills/<name>/SKILL.md`, `~/.claude/skills/<name>/SKILL.md`, `.agents/skills/<name>/SKILL.md`. Missing refs: warn and continue.
6. **Fetch shared context** — diff, blame, in-code comments, past-PR comments (see "Shared review context" above).
7. **Spawn parallel section agents** — one per section, each with its `model`, its section body + resolved refs, the shared context, the false-positive list, and the confidence rubric. Each returns findings of shape:
   ```yaml
   path: src/users.ts
   line_start: 45
   line_end: 47
   message: "Missing null check before accessing user.profile.name"
   evidence: "API returns sparse objects; rule under Bugs"
   confidence: 85
   ```
8. **Merge + dedup** — collapse same-line findings across sections (keep the highest confidence).
9. **Filter** — drop findings below `min_confidence`.
10. **Post-eligibility re-check (Haiku)** — race guard. If PR closed mid-review, abort posting and explain.
11. **Open pending review** — `gh pr-review review --start --commit <head-sha> --pr <n>`. Capture the returned `review-id`.
12. **Add inline comments** — for each finding: `gh pr-review review --add-comment --review-id <id> --pr <n> --path <p> --line <end> --start-line <start> --body <message>`. Inline anchors must point at lines in the delta diff (older lines cited for context can appear in the message body, not the anchor).
13. **Summary draft** — compose a markdown summary listing sections checked, finding counts, and citations to specific snippets using the full-SHA permalink format. Header `### Automated review (Claude)`, footer `🤖 Generated by code-review plugin`.
14. **Stop — do not submit.** Print:
    ```
    Pending review created with N inline comments.
    Review on GitHub: <url>
    Submit when ready:
      gh pr-review review --submit --review-id <id> --pr <n> \
        --event COMMENT|REQUEST_CHANGES|APPROVE \
        --body "<summary you can edit>"

    Or submit now? (y/n)
    ```
    If user picks `y`, ask which event, then run the submit command. If `n`, exit.

### `/review-discussion [pr] <question>`

**Purpose:** answer a specific question about a PR, lazily loading only what's needed. Optionally roll the answer into a pending review if the answer surfaces concrete actionable issues.

**Distinct from `/review`:** narrow Q&A rather than comprehensive scan. Sequential single answering agent rather than parallel section agents. No eligibility checks. No delta logic. Free to pull from skills not in the artifact, MCP tools, web, and arbitrary files.

**Examples:**

- "Is the naming on `UserService` in `src/users.ts` good?"
- "Does this PR follow OWASP at the auth boundary?"
- "Is the test coverage on the new module sufficient — compare with `agents/skills/riverpod-best-practices`."
- "Look at the comments on PR #45 we discussed — does this PR address them?"

**Invocation:**

- `/review-discussion <question>` — current branch's PR
- `/review-discussion <pr> <question>` — explicit PR number or URL
- Question is required; PR auto-detects from current branch otherwise.

**Flow:**

1. **Resolve PR.** Bail with usage if neither explicit PR nor a current-branch PR exists.
2. **No eligibility checks.** Closed/merged/draft/already-reviewed-by-self all proceed. The user is asking a specific question; PR state is not a gate.
3. **Lazy-load planner (Haiku).** Inputs: the question text, the artifact frontmatter (refs and section names only, no bodies), and `gh pr view --json files` (file list, no contents). Returns:
   ```yaml
   diff_scope: full | files: ["src/users.ts"] | hunks: ["src/users.ts:40-80"]
   sections_to_load: [conventions, simplicity]
   refs_to_load: ["AGENTS.md", "skill:code-simplifier"]
   extra_resources:
     skills: ["riverpod-best-practices"]   # mentioned in question, not in refs
     mcp_tools: []
     web: false
     files: ["src/auth.ts"]                # specific paths cited in question
   ```
4. **Targeted loads.** Only the planner-listed files/hunks/sections/refs are loaded. Diff is fetched scoped (`gh pr diff <pr> -- <files>` or filtered to hunks). Section bodies and resolved refs are loaded. Extra resources (off-artifact skills, MCP tools, files) are loaded as listed.
5. **Answering agent (sequential, model `inherit` by default).** Single agent receives the question, the loaded scope, and a system instruction that includes the false-positive baseline (so it does not over-flag). Agent answers in prose, citing specific lines/files using the full-SHA permalink format when grounding in PR code. Agent may call WebFetch / WebSearch / MCP tools / Read / Grep / Bash as needed.
6. **Post-answer disposition.** After the answer is printed, if the agent surfaced concrete actionable issues (its own judgment), prompt:
   ```
   I'd suggest <N> inline comments based on this. Draft a pending review? (y/n)
   ```
   On `y`, re-use the `/review` posting machinery (steps 11–14: open pending, add inline comments anchored to specific lines, do not submit). The user can also pre-empt this by including instructions in the question itself ("…and draft a pending review for what you find").
7. **Failure modes:**
   - PR doesn't exist → bail with usage hint.
   - PR has no diff (empty PR) → answer based on PR metadata only; warn.
   - Best-practices artifact missing → answer using ad-hoc reasoning plus any refs/skills the user mentioned in the question; warn that artifact-grounded reasoning is unavailable.

**Comparison: `/review` vs. `/review-discussion`:**

| Aspect | `/review` | `/review-discussion` |
|--------|-----------|---------------------|
| Trigger | "review this PR" | "answer this question" |
| Scope decision | All sections, full diff | Planner picks sections + diff scope from question |
| Eligibility | Strict (closed/merged/draft/already-reviewed) | None |
| Delta logic | Yes (only new commits since last review) | No |
| Architecture | Parallel section agents | Sequential answering agent |
| Default action | Open pending review with inline comments | Print answer; ask before drafting any review |
| External sources | Refs only | Refs + ad-hoc skills / MCP / web / files |
| Default model | Per-section frontmatter | `inherit` |

## UX defaults (locked in)

| Question | Default |
|----------|---------|
| Auto-discover docs/skills during init | on |
| Discussion command loop | one rule per invocation, ask "another?" with default `n` |
| PR-insight confirmation | per-rule walkthrough (`select`) |
| Mine review summary bodies as well as inline | yes |
| Submit review automatically | **never** |

## Open questions / out of scope

- **Project-specific false-positives in frontmatter** — could add a `false_positives:` list. Not in v1; revisit if users hit project-specific noise.
- **Per-finding standalone confidence-scoring agent** — the existing `/code-review` plugin runs a separate Haiku agent per finding. We use single-pass instead (section agent returns its own confidence) to halve cost. Worth re-evaluating only if confidence quality looks bad.
- **Skill bundling for plugin-internal helpers** — held off. If multiple commands end up duplicating the same logic (rule extraction, frontmatter parsing, ref resolution), promote to skills under `plugins/code-review/skills/`.
- **Public marketplace distribution** — out of scope. Personal plugin only.
- **Codex/Cursor parity** — out of scope. Claude Code only.

## Implementation notes

- Each command file is a Claude Code slash-command markdown with frontmatter (`allowed-tools`, `description`). The bulk of the prompt encodes the flow.
- `gh pr-review` is a hard dependency. The plugin should detect missing extension (`gh extension list | grep pr-review`) on first use and print the install command: `gh extension install agynio/gh-pr-review`.
- YAML frontmatter parsing: keep it simple — use `yq` (CLI) if available, otherwise fall back to a manual front-matter regex split since the schema is small and stable.
- The `inherit` model in section frontmatter means "use whatever model the parent agent is running" — at command time, this is the model of the command-invoking session, passed through to spawned subagents unless overridden.
- Skill resolution must support both project-local (`.claude/skills/`, `.agents/skills/`) and user-global (`~/.claude/skills/`) paths. Use the same precedence Claude Code itself uses.
