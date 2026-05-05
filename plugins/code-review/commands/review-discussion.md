---
allowed-tools: Bash(gh pr-review:*), Bash(gh pr:*), Bash(gh api:*), Bash(gh repo:*), Bash(git:*), Read, Glob, Grep, WebFetch, WebSearch, Agent
description: Answer a focused question about a PR; lazy-load only what's needed; optionally draft a pending review
disable-model-invocation: false
---

You are answering a focused question about a single pull request. Unlike `/review`, this is **narrow Q&A**, not a comprehensive scan. You lazy-load only what the question needs (a planner picks the scope), spawn a single sequential answering agent, print its prose answer, and — only if the answer surfaces concrete actionable issues — optionally roll those into a pending GitHub review.

The arguments passed to this command are: `$ARGUMENTS`.

You have access to the bundled `gh-pr-review` skill (loaded with this plugin). Use it for any pending-review CLI invocation; do not re-document its flags here. Review IDs are `PRR_…`, thread IDs are `PRRT_…`, comment node IDs are `PRRC_…`.

Walk the steps below in order. Do not skip steps. The "Hard rules" block at the bottom is non-negotiable — re-read it before each step.

## Parse args

Strip leading/trailing whitespace from `$ARGUMENTS`. Split on whitespace into tokens.

**Optional `--model X` flag.** Scan the tokens for an adjacent pair `--model <opus|sonnet|haiku>` (case-insensitive on the value). If found, capture `<X>` as the explicit answering-agent model and **remove both tokens** from the argument list before continuing. Anything other than `opus`/`sonnet`/`haiku` after `--model` is a usage error — print the usage block (below) and exit. If no `--model` flag is present, the answering agent inherits from the parent (default).

After flag stripping, treat the remaining tokens as `[<maybe-pr>] <question…>`.

**Two acceptable forms:**

- `<pr> <question>`: the first remaining token is "PR-like", and the rest is the question.
- `<question>`: no PR-like first token; the entire remaining string is the question, and the PR is auto-detected from the current branch.

A token is **"PR-like"** if it matches **either**:

- All-digits (e.g. `42`), or
- A regex match for `https?://github.com/.+/pull/\d+`.

Anything else (e.g. `is`, `does`, `compare`) is **not** PR-like — fall through to the question-only form and treat the entire remaining string as the question.

**Question is required.** If the question is empty after parsing (no remaining tokens, or only the PR token was supplied), print EXACTLY:

```
Usage: /review-discussion [pr] <question>
  pr is optional (URL or number); question is required.
```

Then exit. Do not proceed.

Carry forward: `<pr-token-or-empty>`, `<question>`, `<explicit-model-or-none>`.

## Resolve PR

Apply the same logic as `/review` step 1:

- **URL form** (`https://github.com/<owner>/<repo>/pull/<n>`): parse `<owner>/<repo>` from the path; PR number is the last path segment.
- **Bare integer** (e.g. `42`): the PR number is the token itself; resolve repo with:

  ```bash
  gh repo view --json nameWithOwner -q .nameWithOwner
  ```

- **Empty PR token**: auto-detect from current branch:

  ```bash
  gh pr view --json number -q .number
  ```

  Resolve repo with `gh repo view --json nameWithOwner -q .nameWithOwner`.

If neither an explicit PR nor a current-branch PR resolves, print:

```
Usage: /review-discussion [pr] <question>
No PR found for the current branch and no PR argument provided.
```

Then exit.

If `gh pr view <pr> -R <owner>/<repo>` errors with "not found" (the PR token is non-existent), print the same usage block plus a hint that the PR doesn't exist, and exit.

Carry `<owner>/<repo>` and `<pr-number>` into the next step.

## No eligibility checks. No delta logic.

Skip them entirely. Closed/merged/draft/already-reviewed-by-self all proceed. The user is asking a specific question; PR state is not a gate. (This is the deliberate contrast with `/review`.) Do not call any pre-flight Haiku agent here. Do not compute a delta diff. Whatever scope the planner picks in step 4 is what the answering agent gets.

## Lazy-load planner

Dispatch a **single** `Agent` tool call (a Haiku sub-agent) to pick the load scope. Use:

- `subagent_type: general-purpose`
- `model: haiku`
- `description:` `PR question planner`
- `prompt:` includes:

  1. **The question text** (with `--model X` already stripped, see step 1).

  2. **Artifact frontmatter only.** Test for `.claude/code-review/best-practices.md` first:

     ```bash
     test -f .claude/code-review/best-practices.md && echo OK || echo MISSING
     ```

     If `MISSING`, pass `{}` and tell the planner artifact frontmatter is unavailable (degraded mode — it can still pick `diff_scope` and `extra_resources` from the question alone). If present, parse the YAML between the first two `---` lines:

     ```bash
     awk '/^---$/{i++; next} i==1' .claude/code-review/best-practices.md
     ```

     Pass only `sections:` (slug → `{refs: [...], model: ...}`) — **section names and refs only, no body content**. The planner needs the section names so it can pick which to load; the bodies are loaded later by the orchestrator after the planner returns.

  3. **PR file list only — no contents:**

     ```bash
     gh pr view <pr-number> -R <owner>/<repo> --json files -q '.files[].path'
     ```

     Pass the resulting newline-delimited list. The planner can use `grep` over the file list to spot files matching a class/function name from the question, and trim `diff_scope` accordingly.

  4. **Planner instructions:**

     - Decide `diff_scope`. **Prefer `files:` over `full`** when the question targets specific paths or symbols. **Prefer `hunks:` over `files:`** when the question mentions a class/function and the planner can identify the file (file list grep is allowed). Use `full` only if the question is genuinely PR-wide (e.g. "any obvious issues?" with no path scoping).
     - Decide `sections_to_load`: pick the artifact section slugs whose names match the question's intent (e.g. "OWASP" → `security`; "naming" → `conventions`/`simplicity`). Empty list is fine if no section matches.
     - Decide `refs_to_load`: pull only the refs from the chosen sections; the planner does **not** need to invent new refs.
     - Decide `extra_resources`:
       - `skills:` — skill names mentioned in the question text but not present as `skill:<name>` refs in the artifact (e.g. user types "compare with riverpod-best-practices" → add `riverpod-best-practices`).
       - `mcp_tools:` — names of MCP tools the question explicitly invokes (rare; usually `[]`).
       - `web:` — `true` if the question references "recent docs", "official docs", "current best practices", an external URL, etc. Otherwise `false`.
       - `files:` — arbitrary paths the question cites that aren't already covered by `diff_scope` (e.g. "look at AGENTS.md").

  5. **Output schema (return as a single YAML object, no prose):**

     ```yaml
     diff_scope: full | files: [<paths>] | hunks: ["<path>:<start>-<end>"]
     sections_to_load: [<slugs>]
     refs_to_load: [<paths or skill:<name>>]
     extra_resources:
       skills: [<skill names mentioned in question, not in artifact refs>]
       mcp_tools: []
       web: <bool>
       files: [<arbitrary paths cited in question>]
     ```

The planner is **just a planner**. It picks scope; it does NOT answer the question. The answering agent in step 6 produces the prose answer.

Carry the planner's YAML output into step 5.

## Targeted loads

Following the planner's plan, load only what's needed.

### Diff

- If `diff_scope: full`:

  ```bash
  gh pr diff <pr-number> -R <owner>/<repo>
  ```

- If `diff_scope: files: [...]`:

  ```bash
  gh pr diff <pr-number> -R <owner>/<repo> -- <files>
  ```

- If `diff_scope: hunks: [...]` (entries shaped `<path>:<start>-<end>`):

  Load the per-file diff first via `gh pr diff <pr-number> -R <owner>/<repo> -- <path>` for each unique path, then trim each diff to lines whose target-side line number is in `[start, end]`. The result is the in-scope diff for the answering agent.

If the diff is empty (an empty PR — no code changes), continue with an empty diff and warn the answering agent that the answer must be based on PR metadata only (no code is visible).

### Section bodies

For each slug in `sections_to_load`:

1. Read the artifact body (everything after the closing `---` of the frontmatter).
2. Enumerate every `## ` heading.
3. Slugify each: lowercase, whitespace runs → `_`, drop characters not in `[a-z0-9_]`.
4. Pick the heading whose slugified form equals the section slug. The body region runs from the line after that heading to the line before the next `## ` heading or EOF.
5. If no body heading slugifies to the slug, warn (`section <slug>: no body heading found, using empty body`) and continue with empty content.

If `sections_to_load` is empty, this step is a no-op.

### Refs

For each ref in `refs_to_load`:

- If the ref starts with `skill:<name>`, resolve in **this exact order**, taking the first hit:
  1. `.claude/skills/<name>/SKILL.md`
  2. `.agents/skills/<name>/SKILL.md`
  3. `~/.claude/skills/<name>/SKILL.md`
  4. `~/.claude/plugins/cache/*/*/skills/<name>/SKILL.md` (glob; first match)
- Otherwise treat the ref as a file path **relative to the project root**. Read it directly with the `Read` tool.
- On a miss for any ref (no path resolved, or file unreadable), warn `ref not resolved: <ref>` and continue with empty content for that ref.

### Extra resources

- **Extra skills** (`extra_resources.skills`): resolve via the same `skill:<name>` precedence as above (no `skill:` prefix here — the names came directly from the question). Same on-miss behavior.
- **Extra files** (`extra_resources.files`): read each with the `Read` tool. On miss, warn and continue.
- **MCP tools** (`extra_resources.mcp_tools`): nothing to pre-load; the answering agent calls them directly when needed.
- **Web** (`extra_resources.web`): nothing to pre-load; the answering agent calls `WebFetch` / `WebSearch` when needed.

Bundle every loaded resource into a labeled blob (each section header naming its source path) for the answering agent.

### Degraded mode (no artifact)

If the artifact was missing in step 4, sections, refs, and section-derived extras are all empty. Warn:

```
Warning: no best-practices.md found at .claude/code-review/. Answering without project rules — artifact-grounded reasoning is unavailable.
```

The answering agent still runs; it just lacks project-specific rules to ground against.

## Answer

Spawn a **single sequential** `Agent` tool call:

- `subagent_type: general-purpose`
- `description:` `Answer PR question`
- `model:` —
  - if step 1 captured an explicit `--model X` (`opus`/`sonnet`/`haiku`), pass that string verbatim;
  - **otherwise omit the `model` parameter entirely** so the answering agent inherits from the parent. (The Agent tool inherits the parent's model when the parameter is absent — this is the default behavior for `/review-discussion`.)
- `prompt:` includes, in this order:

  1. **The question text** (with `--model X` stripped).

  2. **PR context:**

     ```
     # PR
     <owner>/<repo> #<pr-number>
     URL: https://github.com/<owner>/<repo>/pull/<pr-number>
     ```

  3. **The in-scope diff** (from step 5). Label it clearly:

     ```
     # Diff
     <diff verbatim, or "(empty PR — no code changes)" if applicable>
     ```

  4. **Loaded section bodies** (from step 5):

     ```
     # Section: <Title>
     <body verbatim>
     ```

     Repeat for each loaded section. Omit this block entirely if no sections were loaded.

  5. **Reference materials** (resolved refs, each labeled with its source path):

     ```
     # Reference materials

     ## <ref-path-1>
     <content>

     ## <ref-path-2>
     <content>
     ```

     Omit if empty.

  6. **Extra resources** (extra skills, extra files), each labeled:

     ```
     # Extra resources

     ## skill:<name> (resolved to <path>)
     <content>

     ## <file-path>
     <content>
     ```

     Omit if empty.

  7. **The false-positive baseline (verbatim, no edits) — these rules tell the answering agent which findings are false positive noise to drop:**

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

     The artifact's `false_positive_filters` (if any) are NOT loaded here; only the baseline above applies.

  8. **Answering instructions:**

     - Answer the question in **prose**. Be direct and grounded.
     - When citing PR code, use the **full-SHA permalink format**:

       ```
       https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<a>-L<b>
       ```

       Use the full HEAD SHA (resolve via `gh pr view <pr> --json headRefOid -q .headRefOid` if needed). Single-line citations use `#L<a>` (no `-L<b>`).
     - You may call `WebFetch`, `WebSearch`, MCP tools, `Read`, `Grep`, and `Bash` (for read-only `gh`/`git` queries) as needed to ground the answer. The pre-loaded scope is a starting point, not a cap; pull more if the question demands it.
     - **Do not run build, typecheck, or lint.** CI handles those.
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

When the answering agent returns, capture its full prose response (including the `### Suggested findings` block, if present).

## Print the answer

Display the agent's prose answer to the user verbatim. This is the primary deliverable — the user always gets the answer, regardless of what comes next.

## Optional pending review

Parse the agent's response. If it contains a `### Suggested findings` block with **at least one** finding, prompt:

```
I'd suggest <N> inline comments based on this. Draft a pending review? (y/n)
```

Where `<N>` is the count of `suggested_findings` entries.

If the response has no `### Suggested findings` block (or the block is empty), **skip this step entirely** — the answer in step 7 stands and the run is complete.

Parse the user's reply (case-insensitive single character; empty defaults to `n`):

- `n` (or empty): exit cleanly. The answer in step 7 stands.
- `y`: continue with the posting flow below.

### Posting flow (on `y`)

1. **Get HEAD SHA:**

   ```bash
   gh pr view <pr-number> -R <owner>/<repo> --json headRefOid -q .headRefOid
   ```

2. **Verify `gh pr-review` extension is installed:**

   ```bash
   gh extension list 2>/dev/null | grep -q pr-review || { echo 'Missing dependency: gh extension install agynio/gh-pr-review'; exit 1; }
   ```

   If missing, print the install command and exit.

3. **Open the pending review:**

   ```bash
   gh pr-review review --start \
     --pr <pr-number> -R <owner>/<repo> \
     --commit <head-sha>
   ```

   Capture the returned `review-id`. It looks like `PRR_…` — that is the literal prefix the `gh-pr-review` skill documents. Carry it as `<review-id>` for the rest of this step.

4. **Add inline comments.** Note: the bundled `gh-pr-review` skill snapshot doesn't document `--start-line`, but the actual CLI supports it for multi-line ranges — run `gh pr-review review --add-comment --help` to confirm the flag is available on the installed version.

   For each entry in `suggested_findings`, run:

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

   - Header line is literally `[discussion · confidence <N>] <message>` (the section is always `discussion` here — there are no parallel section agents).
   - `<N>` is the finding's `confidence`.
   - `<message>` is the finding's `message`.
   - `<evidence>` is the finding's `evidence`.
   - **If `line_end == line_start`, omit `--start-line`** (single-line comments don't take a range).
   - Anchors must be inside the diff loaded in step 5. The answering agent was instructed to keep them in range; if a single anchor still fails, log `failed to anchor: <path>:<start>-<end>` and continue with the next finding (do not abort the whole posting flow on a single anchor failure).

5. **Do NOT submit.** Print EXACTLY (substituting captured values):

   ```
   Pending review created with <N> inline comments.
   GitHub URL: <pr-url>
   Submit when ready:
     gh pr-review review --submit --review-id <review-id> --pr <pr-number> -R <owner>/<repo> \
       --event COMMENT|REQUEST_CHANGES|APPROVE \
       --body "<your summary>"

   Submit now? (y/n)
   ```

   Where `<pr-url>` is `https://github.com/<owner>/<repo>/pull/<pr-number>` and `<N>` is the count of inline comments actually posted (after subtracting any that failed to anchor in step 4).

6. **Parse the user's submit reply** (case-insensitive single character; empty defaults to `n`):

   - `n` (or empty): exit cleanly. The pending review remains visible on GitHub for manual editing or submission later.
   - `y`: ask EXACTLY:

     ```
     Event? (COMMENT/REQUEST_CHANGES/APPROVE)
     ```

     Validate the reply against the three options (case-insensitive). On invalid input, echo what was received and re-ask:

     ```
     Got "<received>", which is not COMMENT/REQUEST_CHANGES/APPROVE. Try again.
     Event? (COMMENT/REQUEST_CHANGES/APPROVE)
     ```

     Once a valid event is captured, run:

     ```bash
     gh pr-review review --submit \
       --review-id <review-id> \
       --pr <pr-number> -R <owner>/<repo> \
       --event <chosen-event> \
       --body "<your summary>"
     ```

     The user's summary body is whatever they typed at the previous step — the orchestrator does not synthesize one for `/review-discussion` (this differs from `/review`, which composes a structured summary). If the user did not provide a summary, send an empty `--body ""` and surface the `gh pr-review` result.

     Surface the result. If submission fails, print the error verbatim and keep the pending review (the user can submit manually).

After this step the run is complete. Do not loop.

## Failure modes

- **PR doesn't exist** (the explicit token resolves to a non-existent PR): bail with the usage hint from step 2 and exit. Do not invent a PR.
- **PR has no diff (empty PR):** continue with an empty diff and warn the answering agent that no code changes are visible. The answer must be based on PR metadata only.
- **Best-practices artifact missing:** planner runs with empty frontmatter `{}`; answering agent runs with no project rules. Print the warning from step 5's "Degraded mode" subsection. Do **not** block the answer.
- **Single ref unresolved:** warn `ref not resolved: <ref>`, continue with empty content for that ref. Do not abort.
- **Single inline-comment anchor failure during posting:** log `failed to anchor: <path>:<start>-<end>`, continue with the next finding.
- **`gh pr-review` extension missing during posting:** print the install command and exit. The answer in step 7 still stood; only the optional draft-review path fails.

## Hard rules

These constraints are non-negotiable. Re-read them before each step:

- **No eligibility checks. No delta logic.** The user's question is the contract, not the PR state. Closed/merged/draft/already-reviewed-by-self all proceed.
- **Default model is `inherit`** (omit the `model` parameter on the answering `Agent` call). Only override on explicit `--model opus`/`--model sonnet`/`--model haiku` in `$ARGUMENTS`. Do not pass the literal string `inherit` — the Agent tool does not accept that value.
- **Never auto-submit a pending review** — even when the user pre-emptively asks in the question (e.g. "draft a pending review for what you find" or "submit it"). Always stop at the "Submit now?" prompt and wait for an explicit `y` plus a chosen event. **Stop. Do not submit.**
- **Never anchor an inline comment to a line outside the diff loaded in step 5.** The answering agent was told to keep `suggested_findings` inside the loaded scope; the orchestrator must enforce this when posting. Drop any finding whose anchor falls outside.
- **The planner is just a planner.** It picks scope; it does NOT answer the question. The answering agent is what produces the prose answer.
- **Use the bundled `gh-pr-review` skill semantics for any pending-review actions.** Do not re-document the CLI flags inline; lean on the skill. Review IDs are `PRR_…`; thread IDs are `PRRT_…`; comment node IDs are `PRRC_…`.
- **If the artifact is missing, do not block the answer.** Proceed in degraded mode with the warning. Artifact absence is an annotation on the answer's grounding, not a stopping condition.
- **Use only the declared tools:** `Bash` (scoped to `gh pr-review:*`, `gh pr:*`, `gh api:*`, `gh repo:*`, `git:*`), `Read`, `Glob`, `Grep`, `WebFetch`, `WebSearch`, `Agent`. Do not request anything else.
