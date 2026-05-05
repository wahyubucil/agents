# Code Review Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a personal Claude Code plugin (`code-review`) that captures project-specific review knowledge in a single committed artifact and applies it to pull requests via parallel section-aware agents.

**Architecture:** A local-marketplace plugin at `agents/plugins/code-review/`. Five slash commands operate over `.claude/code-review/best-practices.md` in target projects. The artifact carries YAML frontmatter (per-section model + refs) and Markdown body (rules). The plugin vendors the upstream `agynio/gh-pr-review` skill so prompts can lean on it instead of re-documenting the CLI.

**Tech Stack:** Markdown slash-command files (Claude Code), YAML frontmatter, `gh` CLI + `agynio/gh-pr-review` extension, Bash, parallel `Agent` tool dispatch.

**Spec:** [docs/superpowers/specs/2026-05-05-code-review-plugin-design.md](../specs/2026-05-05-code-review-plugin-design.md)

**Authorship convention:** This plan tells you *what* to build and *how to verify it*. For code and config (manifests, templates, scripts), full content is shown. For slash-command prompt prose, the plan specifies the contract — the inputs, outputs, exact gh/`Agent` commands to embed, prompts to embed verbatim (false-positive list, confidence rubric), section structure, and concrete smoke-test scenarios. Authorship of the prose happens during the task; verification is the smoke test.

---

## Phase 0: Plugin scaffolding & marketplace

**Files:**
- Create: `.claude-plugin/marketplace.json` (repo root)
- Create: `plugins/code-review/.claude-plugin/plugin.json`
- Create: `plugins/code-review/README.md` (skeleton)
- Create: `plugins/code-review/.gitkeep` placeholders for `commands/`, `skills/`, `templates/` if needed

### Task 0.1: Create the marketplace registration

- [ ] **Step 1: Create the marketplace directory and file**

```bash
mkdir -p /Users/wahyubucil/projects/github.com/wahyubucil/agents/.claude-plugin
```

Then write `/Users/wahyubucil/projects/github.com/wahyubucil/agents/.claude-plugin/marketplace.json`:

```json
{
  "name": "wahyubucil-agents",
  "description": "Personal agents, skills, and plugins for Claude Code",
  "owner": {
    "name": "Wahyu Budi Saputra",
    "email": "wahyubucil@gmail.com"
  },
  "plugins": [
    {
      "name": "code-review",
      "description": "Project-aware code review with per-section parallel agents and pending GitHub reviews",
      "version": "0.1.0",
      "source": "./plugins/code-review",
      "author": {
        "name": "Wahyu Budi Saputra",
        "email": "wahyubucil@gmail.com"
      }
    }
  ]
}
```

- [ ] **Step 2: Verify JSON parses**

Run: `python3 -c "import json; json.load(open('/Users/wahyubucil/projects/github.com/wahyubucil/agents/.claude-plugin/marketplace.json'))"`
Expected: no output (success).

### Task 0.2: Create the plugin manifest

- [ ] **Step 1: Create the plugin's `.claude-plugin/plugin.json`**

```bash
mkdir -p /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/.claude-plugin
mkdir -p /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/commands
mkdir -p /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills
mkdir -p /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/templates
```

Then write `/Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/.claude-plugin/plugin.json`:

```json
{
  "name": "code-review",
  "description": "Project-aware code review with per-section parallel agents and pending GitHub reviews. Five slash commands: /init, /gather-insight-discussion, /gather-insight-pr, /review, /review-discussion.",
  "version": "0.1.0",
  "author": {
    "name": "Wahyu Budi Saputra",
    "email": "wahyubucil@gmail.com"
  }
}
```

- [ ] **Step 2: Verify plugin.json parses**

Run: `python3 -c "import json; json.load(open('/Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/.claude-plugin/plugin.json'))"`
Expected: no output.

### Task 0.3: Create README skeleton

- [ ] **Step 1: Write `plugins/code-review/README.md`**

```markdown
# code-review plugin

Project-aware code review with per-section parallel agents and pending GitHub reviews.

> Detailed README is filled in at Phase 8 (after all commands are implemented). This is a placeholder.

## Status

In development. Tracking implementation plan: `docs/superpowers/plans/2026-05-05-code-review-plugin.md`.
```

### Task 0.4: Smoke-test plugin install

- [ ] **Step 1: Install the marketplace and plugin in Claude Code**

In a Claude Code session at `/Users/wahyubucil/projects/github.com/wahyubucil/agents`:
```
/plugin marketplace add ./
/plugin install code-review@wahyubucil-agents
```
Expected: marketplace registers, plugin installs without error.

- [ ] **Step 2: Verify the plugin is listed**

```
/plugin list
```
Expected: `code-review@wahyubucil-agents` appears as installed.

- [ ] **Step 3: Commit Phase 0**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  .claude-plugin/marketplace.json \
  plugins/code-review/.claude-plugin/plugin.json \
  plugins/code-review/README.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): scaffold plugin and marketplace registration"
```

---

## Phase 1: Vendor the gh-pr-review skill

**Files:**
- Create: `plugins/code-review/skills/gh-pr-review/SKILL.md`
- Create: `plugins/code-review/skills/gh-pr-review/LICENSE`
- Create: `plugins/code-review/skills/gh-pr-review/VERSION`

**Pin:** upstream commit `d2c86f61c5709c567e8487ba82f04112a456c221` on `main` (dated 2026-04-13). The latest tagged release `v1.6.2` (2025-12-08) predates both `SKILL.md` (added 2026-01-28) and `LICENSE` (added 2026-04-12), so we pin to a commit on `main` until upstream cuts a tag covering both. Re-vendor by changing this commit SHA and re-running the fetch.

### Task 1.1: Vendor the skill

- [ ] **Step 1: Fetch SKILL.md at the pinned commit and write locally**

Run (the URL must be quoted because of the `?` glob char in zsh):
```bash
gh api 'repos/agynio/gh-pr-review/contents/SKILL.md?ref=d2c86f61c5709c567e8487ba82f04112a456c221' --jq '.content' \
  | base64 -d > /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills/gh-pr-review/SKILL.md
```
Expected blob SHA: `2587753d0d99a2732a749d5aef7bebd5231dcf62` (5329 bytes).

- [ ] **Step 2: Fetch LICENSE at the pinned commit and write locally**

```bash
gh api 'repos/agynio/gh-pr-review/contents/LICENSE?ref=d2c86f61c5709c567e8487ba82f04112a456c221' --jq '.content' \
  | base64 -d > /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills/gh-pr-review/LICENSE
```
Expected blob SHA: `67a7464e95e37a9a00044d65cac56c3280768266` (1061 bytes).

- [ ] **Step 3: Write VERSION file**

Write `/Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills/gh-pr-review/VERSION`:

```
upstream: agynio/gh-pr-review
commit: d2c86f61c5709c567e8487ba82f04112a456c221
commit_date: 2026-04-13
fetched_at: 2026-05-05
note: pinned to a commit on main; latest release tag v1.6.2 predates SKILL.md and LICENSE
```

- [ ] **Step 4: Verify SKILL.md frontmatter is valid**

Run:
```bash
head -5 /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills/gh-pr-review/SKILL.md
```
Expected: starts with `---`, `name: gh-pr-review`, `description: ...`, `---` — confirms frontmatter is intact.

- [ ] **Step 5: Verify LICENSE is the MIT text**

Run:
```bash
head -3 /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/skills/gh-pr-review/LICENSE
```
Expected: starts with `MIT License`.

### Task 1.2: Smoke-test skill auto-loading

- [ ] **Step 1: Restart the Claude Code session (or `/plugin reload`)** to pick up new skill files inside the installed plugin.

- [ ] **Step 2: Confirm the skill is discoverable**

Ask Claude: "List skills available from the code-review plugin." Or check `/skill list` output.
Expected: `gh-pr-review` skill appears, sourced from the plugin.

- [ ] **Step 3: Commit Phase 1**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/skills/gh-pr-review/SKILL.md \
  plugins/code-review/skills/gh-pr-review/LICENSE \
  plugins/code-review/skills/gh-pr-review/VERSION
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): vendor agynio/gh-pr-review skill (commit d2c86f6)"
```

---

## Phase 2: Best-practices artifact template

**Files:**
- Create: `plugins/code-review/templates/best-practices.template.md`

The template is what `/init` writes (with prompt-time substitutions). It must round-trip through standard YAML parsing.

### Task 2.1: Author the template

- [ ] **Step 1: Write the template file**

Write `/Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/templates/best-practices.template.md`:

````markdown
---
sections:
  security:
    refs: []
    model: inherit
  bugs:
    refs: []
    model: inherit
  performance:
    refs: []
    model: inherit
  simplicity:
    refs: []
    model: inherit
  testing:
    refs: []
    model: inherit
  error_handling:
    refs: []
    model: inherit
  conventions:
    refs: []
    model: inherit

min_confidence: 80
skip_authors:
  - dependabot[bot]
  - renovate[bot]
  - github-actions[bot]
---

# Code Review Best Practices

This file is consumed by the `code-review` plugin (see `/init`, `/review`, `/review-discussion`).

- Frontmatter is **machine-readable**: per-section `refs` (file paths or `skill:<name>`) and `model` (`inherit`/`sonnet`/`opus`/`haiku`).
- Body is **human-readable**: rules per section as bullets, with optional `**Why:**` and `**Example:**` sub-bullets.

## Rule format

```markdown
- **Always check for null** before accessing nested properties on API responses
  - **Why:** API returns sparse objects when fields are unauthorized
  - **Example:** `user?.profile?.name ?? 'Anonymous'`
```

## Security
<!-- Add inline rules here, or rely on refs above -->

## Bugs
<!-- Add inline rules here, or rely on refs above -->

## Performance
<!-- Add inline rules here, or rely on refs above -->

## Simplicity
<!-- Add inline rules here, or rely on refs above -->

## Testing
<!-- Add inline rules here, or rely on refs above -->

## Error Handling
<!-- Add inline rules here, or rely on refs above -->

## Conventions
<!-- Add inline rules here, or rely on refs above -->
````

### Task 2.2: Verify the template

- [ ] **Step 1: Verify YAML frontmatter parses**

Run:
```bash
awk '/^---$/{i++; next} i==1' /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/templates/best-practices.template.md \
  | python3 -c "import sys, yaml; print(yaml.safe_load(sys.stdin))"
```
Expected: prints a Python dict containing `sections`, `min_confidence: 80`, `skip_authors: [...]`. No parse errors. (If `yaml` is missing: `pip3 install pyyaml`.)

- [ ] **Step 2: Verify section count and slugs**

Run:
```bash
awk '/^---$/{i++; next} i==1' /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/templates/best-practices.template.md \
  | python3 -c "import sys, yaml; d=yaml.safe_load(sys.stdin); s=list(d['sections'].keys()); assert s==['security','bugs','performance','simplicity','testing','error_handling','conventions'], s; print('OK')"
```
Expected: `OK`.

- [ ] **Step 3: Verify body section headings match slugs**

Run:
```bash
grep -E '^## ' /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/templates/best-practices.template.md
```
Expected output (in order):
```
## Security
## Bugs
## Performance
## Simplicity
## Testing
## Error Handling
## Conventions
```

- [ ] **Step 4: Commit Phase 2**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/templates/best-practices.template.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add best-practices artifact template"
```

---

## Phase 3: `/init` command

**Files:**
- Create: `plugins/code-review/commands/init.md`

**Contract** (what the command does — encode this in the prompt):

| Aspect | Behavior |
|--------|----------|
| Args | `$ARGUMENTS` may contain `--quick` to skip prompts |
| Existence | If `.claude/code-review/best-practices.md` exists, prompt `(r)eplace / (m)erge / (a)bort`; default abort |
| Auto-discovery | Probe target repo & skill paths; surface suggestions per section |
| Section walk | For each of 7 default sections + custom: ask refs + model |
| Globals | Confidence threshold (80), skip-authors |
| Quick mode | Skip all prompts; defaults from template |
| Output | Write file at `.claude/code-review/best-practices.md` |

**Auto-discovery probes:**

| Section | File patterns | Skill name patterns |
|---------|---------------|---------------------|
| `security` | `SECURITY.md`, `docs/security/**`, files with `security` in name | any installed skill matching `*security*`, `owasp` |
| `conventions` | `CONTRIBUTING.md`, `STYLE.md`, `AGENTS.md`, root `CLAUDE.md` | n/a (project conventions tend to be docs not skills) |
| `performance` | `docs/performance/**`, files with `perf` or `performance` in name | any installed skill matching `*performance*`, `*perf*` |
| `simplicity` | n/a (most projects don't have a simplicity doc) | `code-simplifier`, anything matching `*simplif*` |
| `testing` | `docs/testing/**`, `TESTING.md` | anything matching `*test*` (excluding `unit-test`) |
| `error_handling` | n/a | n/a |
| `bugs` | n/a | n/a |

**Skill probe paths** (for the target project being initialized):
- `.claude/skills/`
- `.agents/skills/`
- `~/.claude/skills/`
- `~/.claude/plugins/cache/*/*/skills/` (installed plugin skills)

### Task 3.1: Author `/init` command file

- [ ] **Step 1: Write the command file with frontmatter**

Write `/Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/commands/init.md` with this frontmatter:

```yaml
---
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
description: Scaffold .claude/code-review/best-practices.md for the current project
disable-model-invocation: false
---
```

- [ ] **Step 2: Author the prompt body**

The body must encode the full contract above. Required structural elements:

- Pick up `--quick` from `$ARGUMENTS`; if present, skip all prompts and use defaults.
- Existence check using Bash: `test -f .claude/code-review/best-practices.md`. On found, prompt and parse single-letter answer (r/m/a, default a).
- For each default section, run the auto-discovery probes (Bash `ls`/`glob`/`find` for file patterns, list skill directories) and surface suggestions in the prompt: `Found: SECURITY.md, docs/security/auth.md. Use as refs? Or specify others.`
- Walk sections in order: `security, bugs, performance, simplicity, testing, error_handling, conventions`. For each: print suggestions, ask `Refs?`, ask `Model?`.
- After defaults, loop on `Add additional section? (name or done)`. For each custom section, ask refs+model; slugify the name (lowercase, spaces→underscores).
- Ask `Confidence threshold? (default 80)`.
- Ask `Skip PR authors? (default: dependabot[bot], renovate[bot], github-actions[bot])`. Accept comma-separated additions.
- Render the final file content from the template at `${CLAUDE_PLUGIN_ROOT}/templates/best-practices.template.md`, substituting frontmatter values per user input. Body sections: keep the `<!-- Add inline rules here, or rely on refs above -->` placeholders (they get replaced as `/gather-insight-*` adds rules).
- Use `Write` to save to `.claude/code-review/best-practices.md` (ensure `.claude/code-review/` exists first).
- Print summary: `Created .claude/code-review/best-practices.md with N sections. Run /gather-insight-discussion or /gather-insight-pr to populate rules.`

The prompt **must** explicitly state: do not invent additional sections beyond the user-provided ones; do not auto-fill rule bodies.

### Task 3.2: Smoke-test `/init`

- [ ] **Step 1: Set up a sandbox project**

```bash
mkdir -p /tmp/code-review-sandbox && cd /tmp/code-review-sandbox && git init -q && touch SECURITY.md CONTRIBUTING.md
```

- [ ] **Step 2: Test `--quick` mode**

In a Claude Code session at `/tmp/code-review-sandbox`:
```
/init --quick
```
Expected:
- File created at `.claude/code-review/best-practices.md`.
- Frontmatter has all 7 default sections, each with `refs: []` and `model: inherit`.
- `min_confidence: 80`, `skip_authors` matches default list.
- No prompts asked.

Verify: `cat /tmp/code-review-sandbox/.claude/code-review/best-practices.md`.

- [ ] **Step 3: Test auto-discovery in interactive mode**

Reset: `rm -rf /tmp/code-review-sandbox/.claude`. Then:
```
/init
```
Expected:
- For `security` section, the prompt mentions `SECURITY.md` as a discovered suggestion.
- For `conventions` section, the prompt mentions `CONTRIBUTING.md` as a discovered suggestion.
- Walk through accepting suggestions and `inherit` model. Final file reflects accepted refs.

- [ ] **Step 4: Test existing-file handling**

Re-run `/init`. Expected: prompt `(r)eplace / (m)erge / (a)bort`. Test all three branches (run thrice, resetting between).

- [ ] **Step 5: Test custom section addition**

Run `/init` again, accept defaults, then at custom-section prompt enter `Accessibility`. Expected: an 8th section `accessibility` (slugified) appears in frontmatter and `## Accessibility` heading appears in body.

- [ ] **Step 6: Commit Phase 3**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/commands/init.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add /init command"
```

---

## Phase 4: `/gather-insight-discussion` command

**Files:**
- Create: `plugins/code-review/commands/gather-insight-discussion.md`

**Contract:**

| Aspect | Behavior |
|--------|----------|
| Args | `$ARGUMENTS` is the optional seed topic |
| Pre-check | If `.claude/code-review/best-practices.md` missing, suggest `/init` and bail |
| Dialogue | Clarifying questions until rule is one clear line + optional `Why:` / `Example:` |
| Section assignment | Propose section based on rule content; user can override or create new |
| Conflict | Semantic similarity scan against existing rules in target section; prompt `(r)eplace / (m)erge / (k)eep both / (s)kip` |
| Confirm | Show diff of new lines under target heading; `(y)es / (e)dit / (n)o` |
| Append | Write into the section's bullet list, preserving existing rules |
| Loop | Ask `Another? (y/n)` default `n` |

**Rule format (must match Phase 2 template):**
```markdown
- **<one-line rule>**
  - **Why:** <reason>
  - **Example:** <example, optional>
```

### Task 4.1: Author `/gather-insight-discussion` command file

- [ ] **Step 1: Write frontmatter**

```yaml
---
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
description: Distill a rule from a discussion and append it to the best-practices file
disable-model-invocation: false
---
```

- [ ] **Step 2: Author the prompt body**

Required structural elements:

- Read `.claude/code-review/best-practices.md`. If missing: print "Run /init first" and exit.
- If `$ARGUMENTS` is non-empty, treat as seed topic; otherwise prompt: `What rule do you want to formalize?`.
- Run a focused dialogue. Ask only the questions necessary to:
  1. Capture the rule as one clear sentence in imperative form.
  2. Capture the **Why** (one sentence, mandatory — non-negotiable).
  3. Capture an **Example** (optional, ask but accept skip).
- Identify target section. Propose one based on rule content. Show the user: `Target section: bugs (or override: security/performance/simplicity/testing/error_handling/conventions/<custom>/<new>)`.
- If user says `<new>`, ask new section name; slugify and add to frontmatter (`refs: []`, `model: inherit`) and add a new `## <Name>` heading at the bottom of body.
- Conflict scan: for each existing bullet under the target section, compare semantically (use your judgment based on text similarity + intent). Surface up to 3 closest matches with line numbers. Prompt: `Found similar rule(s): ... (r)eplace / (m)erge / (k)eep both / (s)kip`.
  - replace: edit the existing rule's text in place.
  - merge: combine into a single rule (ask user to confirm merged text).
  - keep both: append as a new bullet.
  - skip: bail out, no change.
- Print the diff (just the affected lines) in a fenced code block. Prompt: `Apply? (y)es / (e)dit / (n)o`. On `e`, let the user edit the rule text inline.
- Use `Edit` to apply the change to `.claude/code-review/best-practices.md`. Append under the section heading; preserve placeholder comment by inserting *above* it if it's still there, or remove the placeholder once the section has at least one rule.
- After write, print: `Added 1 rule to <section>.` Then ask `Another? (y/n)` (default n). On y, loop from the topic prompt.

The prompt **must** instruct: never silently rewrite existing rules; never auto-merge without user confirmation; never write a rule without a `**Why:**`.

### Task 4.2: Smoke-test `/gather-insight-discussion`

- [ ] **Step 1: Continue with `/tmp/code-review-sandbox`**

Ensure `.claude/code-review/best-practices.md` exists (from Phase 3). Reset if needed via `/init --quick`.

- [ ] **Step 2: Test bare invocation**

```
/gather-insight-discussion
```
Expected:
- Asks for topic.
- Asks for Why (mandatory).
- Asks for Example (optional, skippable).
- Suggests a section, allows override.
- Shows diff.
- After `y`, applies the edit.
- Asks `Another? (y/n)` default `n`.

Verify: `cat /tmp/code-review-sandbox/.claude/code-review/best-practices.md` shows the new bullet under the chosen section, with `**Why:**` sub-bullet.

- [ ] **Step 3: Test seed topic from `$ARGUMENTS`**

```
/gather-insight-discussion always validate input at trust boundaries
```
Expected: dialogue starts with this seed already loaded; agent jumps to asking for Why.

- [ ] **Step 4: Test conflict detection**

Add a rule like "always check user input for nulls" via `/gather-insight-discussion`. Then run again with "always validate user input is not null". Expected: conflict prompt fires; user can pick replace/merge/keep both/skip; result matches choice.

- [ ] **Step 5: Test new-section creation**

In the dialogue, when prompted for target section, enter `<new>` then `Accessibility`. Expected: `accessibility` slug added to frontmatter, `## Accessibility` heading appended at bottom of body, rule placed under it.

- [ ] **Step 6: Commit Phase 4**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/commands/gather-insight-discussion.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add /gather-insight-discussion command"
```

---

## Phase 5: `/gather-insight-pr` command

**Files:**
- Create: `plugins/code-review/commands/gather-insight-pr.md`

**Contract:**

| Aspect | Behavior |
|--------|----------|
| Args | `$ARGUMENTS` = PR URL or number; required |
| Pre-check | Best-practices file must exist |
| Fetch | `gh pr-review review view <pr> --include-comment-node-id` (single call) |
| Author filter | Drop bots (`[bot]` suffix or built-in list); **keep self** |
| Substantive filter | Drop short / approval-only / emoji-only |
| Extraction | Parallel sub-agents (Sonnet) per surviving comment → structured rule |
| Generalizability | Drop rules with score < 60 (project-specific nits) |
| Dedup | Semantic match against existing rules; bucket new/duplicate/refines |
| Confirm | Print counts; prompt `(a)ccept all / (s)elect / (n)one`; default select walks one at a time |
| Append | Write accepted rules into appropriate sections |

**Built-in bot list (case-insensitive substring match on author login):**
`coderabbit`, `sourcery`, `codiumai`, `codeball`, `codeguru`, `github-actions`, `dependabot`, `renovate`. Plus any `[bot]` suffix. Plus the project's `skip_review_authors` from frontmatter.

**Substantive filter rules:**
- Length < 20 chars → drop.
- Body matches case-insensitive any of: `lgtm`, `looks good`, `looks good to me`, `approved`, `approve`, `nit`, `nit:`, `+1`, `:+1:` → drop (regex word-boundary).
- No alphabetic characters (emoji-only) → drop.

**Extracted rule schema (sub-agent output, YAML or JSON):**
```yaml
rule: <one-line imperative>
why: <one sentence>
evidence: "Reviewer @<login> on <path>:<line> in PR #<n>"
suggested_section: <slug>
generalizability: 0-100
```

### Task 5.1: Author `/gather-insight-pr` command file

- [ ] **Step 1: Write frontmatter**

```yaml
---
allowed-tools: Bash(gh pr-review:*), Bash(gh pr view:*), Bash(gh api:*), Read, Edit, Write
description: Mine human PR review comments and append generalizable rules
disable-model-invocation: false
---
```

- [ ] **Step 2: Author the prompt body**

Required structural elements:

- Parse `$ARGUMENTS`. If empty, print usage and exit. Accept URL or number.
- Read `.claude/code-review/best-practices.md`. If missing: bail with hint to run `/init`. Parse frontmatter to get `skip_review_authors` (if present).
- Resolve repo: parse from URL if URL form, else use `gh repo view --json nameWithOwner`.
- Fetch reviews: `gh pr-review review view <pr> -R <owner>/<repo> --include-comment-node-id`. Capture as JSON.
- Iterate over `reviews[].comments[]` (top-level review summary bodies AND inline thread comments — `gh pr-review` returns both in the structured tree).
- Author filter: drop if `author_login` matches `[bot]` suffix, is in built-in list, or is in `skip_review_authors`. **Do not drop self.**
- Substantive filter: drop if `len(body.strip()) < 20`, or matches LGTM regex, or `re.search(r'[a-zA-Z]', body) is None`.
- Spawn parallel `Agent` calls (`model: sonnet`) — **issue all calls in a single response message** so they run concurrently. One agent per surviving comment. Use `subagent_type: general-purpose` and pass:
  - The comment body, `path`, `line`, `author_login`, PR number/repo for evidence.
  - The list of allowed section slugs (from frontmatter).
  - Instruction: extract a generalizable rule per the schema above, with `generalizability: 0-100` (100 = applies broadly, 0 = one-off rename).
- Collect all extractions. Drop those with `generalizability < 60`.
- Read the existing rules from each target section. Dedup: for each candidate, compare against existing rules in `suggested_section` (semantic, agent's judgment).
  - Bucket: `new`, `duplicate` (drop), `refines-existing` (offer to replace existing).
- Print summary: `Mined N comments from M reviewers. Found: X new, Y refine, Z duplicates (skipped).`
- Prompt: `Review the X new + Y refining rules? (a)ccept all / (s)elect / (n)one`. Default `s`.
- On `select`: walk one at a time. For each: show rule + evidence + bucket; prompt `(y)es / (e)dit / (n)o`. On `e`, allow inline edit.
- Apply accepted rules using `Edit` (preserving rule format and placeholder removal logic from Phase 4).
- Print final summary: `Added K rules across <list of sections> from PR #<n>.`

The prompt **must** explicitly state: do not write rules without a Why; do not lower the generalizability threshold without user instruction; do not invent rules from nothing if a comment is unclear (skip instead).

### Task 5.2: Smoke-test `/gather-insight-pr`

Pick a real PR you have access to with at least one human reviewer comment AND a bot reviewer (e.g. CodeRabbit, dependabot). If you don't have one handy, create a quick test PR in a sandbox repo.

- [ ] **Step 1: Test bot filtering**

```
/gather-insight-pr <pr-url-with-human-and-bot>
```
Expected: summary line shows comments mined from human reviewers only; bot author count = 0 in the "from M reviewers" tally.

- [ ] **Step 2: Test self inclusion**

PR where you (`wahyubucil`) left a substantive review comment on someone else's PR. Run `/gather-insight-pr <url>`. Expected: your comment appears in the extraction set, not filtered out.

- [ ] **Step 3: Test substantive filter**

PR with a thread containing only `LGTM 👍`. Expected: filtered out; not extracted.

- [ ] **Step 4: Test generalizability filter**

PR with a comment like `rename foo to bar`. Expected: extracted but generalizability < 60, dropped.

- [ ] **Step 5: Test dedup**

Run twice in succession on the same PR. Expected: second run shows all rules as duplicates, none added.

- [ ] **Step 6: Test interactive select**

PR with 3+ extractable comments. Expected: walkthrough one at a time; can accept/edit/skip each; final summary reflects choices.

- [ ] **Step 7: Commit Phase 5**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/commands/gather-insight-pr.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add /gather-insight-pr command"
```

---

## Phase 6: `/review` command

**Files:**
- Create: `plugins/code-review/commands/review.md`

**Contract:**

| Aspect | Behavior |
|--------|----------|
| Args | `$ARGUMENTS` = optional PR URL/number; auto-detect from current branch otherwise |
| Pre-flight (Haiku) | Skip closed/merged/draft/skip-author/already-reviewed-at-HEAD |
| Delta logic | If prior review by self exists at older SHA, diff only new commits |
| Artifact | Read frontmatter + body; warn-but-continue if missing |
| Refs resolution | For each section's `refs`, load file or skill content |
| Shared context | Diff + git blame + in-code comments + past PR comments on touched files |
| Section agents | Parallel `Agent` per section; model from frontmatter; return findings with confidence |
| Filter | Drop findings below `min_confidence`; merge dups by line range |
| Post-eligibility | Re-check PR state (Haiku), bail if PR closed mid-review |
| Open pending review | `gh pr-review review --start --commit <head-sha> --pr <n>` |
| Inline comments | `gh pr-review review --add-comment ...` per finding |
| Summary draft | Compose markdown body; do NOT submit |
| User prompt | Print submit command + `Submit now? (y/n)`; on y ask event then submit |

**Embedded prompts (must appear verbatim in section-agent prompt):**

**False-positive baseline:**
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

**Confidence rubric:**
```
0:   false positive, fails light scrutiny, or is pre-existing
25:  might be real, agent could not verify
50:  real but minor or rare in practice
75:  real, important, very likely to bite, or directly cited by a rule
100: real, will happen frequently, evidence directly confirms
```

**Build/typecheck/lint disclaimer:** "Do not run build, typecheck, or lint. CI handles those."

**Citation format (for summary body):** `https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>` with ≥1 line of context.

### Task 6.1: Author `/review` command file

- [ ] **Step 1: Write frontmatter**

```yaml
---
allowed-tools: Bash(gh pr-review:*), Bash(gh pr:*), Bash(gh api:*), Bash(git:*), Read, Glob, Grep, Agent
description: Run a project-aware code review and post a pending GitHub review
disable-model-invocation: false
---
```

- [ ] **Step 2: Author the prompt body**

Required structural elements (in execution order):

1. **Resolve PR.**
   - Parse `$ARGUMENTS`: URL → repo + number; bare number → current repo + number; empty → `gh pr view --json number,headRepository -q .number`.
   - Bail with usage if no PR found and no current-branch PR exists.

2. **Pre-flight eligibility (dispatched as Haiku via `Agent` tool).**
   - Run `gh pr view <pr> --json state,isDraft,author,reviews` (and `headRefOid` for HEAD SHA).
   - Bail with reason if state is CLOSED/MERGED, isDraft is true, or author is in `skip_authors`.
   - Check `reviews[]` for any review by self. If none → full review. If exists → record `prior_review_commit_sha`.

3. **Delta computation.**
   - Use `gh pr-review review view <pr> --reviewer <self> --include-comment-node-id` to get the prior review's commit anchor.
   - HEAD SHA via `gh pr view --json headRefOid -q .headRefOid`.
   - If prior commit == HEAD SHA: bail "already reviewed at HEAD".
   - Else: `git fetch && git diff <prior_commit>..<head_sha>` is in-scope. If empty: bail with the same message.

4. **Load artifact.**
   - Read `.claude/code-review/best-practices.md`. Parse frontmatter (sections, min_confidence, skip_authors).
   - If missing: print warning and proceed in degraded mode (only false-positive baseline + confidence rubric, no project rules; sections dict is empty; review still spawns one default Sonnet agent on the diff with a "no project-specific rules — focus on obvious bugs" prompt).

5. **Resolve refs.** For each section in frontmatter:
   - For each ref:
     - If starts with `skill:<name>`, resolve in this order (project-local before user-global, matching Claude Code's own precedence): `.claude/skills/<name>/SKILL.md` → `.agents/skills/<name>/SKILL.md` → `~/.claude/skills/<name>/SKILL.md` → plugin-shipped skills under `~/.claude/plugins/cache/*/*/skills/<name>/SKILL.md`. Read content from the first hit.
     - Else treat as file path relative to project root. Read content.
     - If neither resolves: warn, continue with empty content for that ref (do not fail the whole review).
   - Read the section body (everything between `## <Section Heading>` and the next `## ` or EOF). Skip the section entirely if its body is only the placeholder comment (`<!-- Add inline rules here, or rely on refs above -->`) AND `refs` is empty — no agent dispatch for inactive sections.
   - Bundle: `{section_slug, model, body, refs_content, ref_paths}`.

6. **Fetch shared context.**
   - Diff: `gh pr diff <pr>` (scoped to delta if applicable).
   - Git blame on changed lines: for each changed hunk, `git blame -L <start>,<end> -- <path>` (truncate output to last 5 unique commits per hunk to keep context small).
   - In-code comments in modified files: `grep -n -E '(TODO|FIXME|NOTE|HACK|@see|@deprecated)' <changed-files>` (only modified files).
   - Past PR comments: `gh pr list --state merged --search "path:<file>" --limit 5 --json number,reviews`. For top 5, fetch their `gh pr-review review view --not_outdated`. Concatenate inline comments.

7. **Spawn parallel section agents.** Issue **all `Agent` tool calls in a single response message** so they execute concurrently (per the Agent tool's parallel-dispatch semantics). For each section bundle:
   - `subagent_type: general-purpose`
   - `model:` — if frontmatter `model` is `sonnet`/`opus`/`haiku`, pass that. If `inherit`, **omit the `model` parameter** (the Agent tool inherits from the parent when the parameter is absent — `inherit` is not a literal value the tool accepts).
   - `description:` short label e.g. `<section> review`.
   - `prompt:` includes:
     - The section heading and body rules.
     - Resolved refs content (under "Reference materials").
     - The diff (delta-scoped).
     - Shared context (blame, in-code comments, past PR comments).
     - The false-positive baseline prompt verbatim (above).
     - The confidence rubric verbatim (above).
     - Build/typecheck/lint disclaimer.
     - Output schema instruction:
       ```yaml
       findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <brief, action-oriented>
           evidence: <which rule or shared context grounds this>
           confidence: <0-100>
       ```
     - "If no findings, return `findings: []`."
   - Collect all section results into a flat list, tagging each with the originating section.

8. **Merge + dedup.** For findings overlapping in `(path, line_start, line_end)` ±2 lines: keep the highest-confidence one; preserve evidence from both in its `evidence` field.

9. **Filter by `min_confidence`.** Drop the rest.

10. **Post-eligibility re-check (Haiku via `Agent`).** Re-run step 2's checks. If state changed, abort posting and explain.

11. **Open pending review:**
    ```bash
    gh pr-review review --start --pr <n> -R <owner>/<repo> --commit <head-sha>
    ```
    Capture the returned `review-id` (PRR_…).

12. **Add inline comments.** For each remaining finding:
    ```bash
    gh pr-review review --add-comment \
      --review-id <id> \
      --pr <n> -R <owner>/<repo> \
      --path <p> \
      --line <line_end> \
      --start-line <line_start> \
      --body "$(cat <<EOF
    [<section> · confidence <N>] <message>

    <evidence>
    EOF
    )"
    ```
    Body format: header line `[<section> · confidence <N>] <one-line action>`, blank line, `<evidence>` paragraph. If `line_end == line_start`, omit `--start-line`. Anchor lines must be in the delta diff; if a finding cites an older line for context, mention it in the body but anchor on a delta line.

13. **Summary body draft.** Compose:
    ```markdown
    ### Automated review (Claude)

    Reviewed sections: <list>.
    Findings: <N> at confidence ≥ <threshold> (from <total candidates> raw).

    Highlights:
    - <link to top finding using full-SHA permalink format with ±1 line context>
    - <…>

    🤖 Generated by code-review plugin (delta vs <prior_commit_short>) — based on `.claude/code-review/best-practices.md`.
    ```
    Use full-SHA permalink format for any cited links.

14. **Stop. Do not submit.** Print:
    ```
    Pending review created with <N> inline comments.
    GitHub URL: <pr-url>
    Submit when ready:
      gh pr-review review --submit --review-id <id> --pr <n> -R <owner>/<repo> \
        --event COMMENT|REQUEST_CHANGES|APPROVE \
        --body <summary above, copy-paste from below>

    <summary body draft>

    Submit now? (y/n)
    ```
    On `y`: ask `Event? (COMMENT/REQUEST_CHANGES/APPROVE)`, then run the submit command.
    On `n`: exit cleanly. The pending review remains visible on GitHub for manual editing.

### Task 6.2: Smoke-test `/review`

Pick a real PR in any of your repos. Ideally one with at least one rule violation in some section.

- [ ] **Step 1: Set up artifact in target repo**

In your target repo: `/init --quick`. Then add a few simple rules via `/gather-insight-discussion` so sections are non-empty.

- [ ] **Step 2: Test current-branch invocation**

On a PR branch:
```
/review
```
Expected:
- Pre-flight passes; PR resolved.
- Section agents spawn in parallel (visible as Agent tool calls).
- Findings appear with confidence scores.
- Pending review opens; inline comments added.
- Output ends with `Submit now? (y/n)` — does NOT auto-submit.
- Choose `n`. Verify on GitHub: the pending review exists with N inline comments, not yet submitted.

- [ ] **Step 3: Test PR number invocation**

```
/review 42
```
Expected: same as above on PR #42 in the current repo.

- [ ] **Step 4: Test PR URL invocation**

```
/review https://github.com/owner/repo/pull/42
```
Expected: same as above, repo derived from URL.

- [ ] **Step 5: Test eligibility skip**

On a closed/merged/draft PR:
```
/review <pr>
```
Expected: bails with reason printed; no review created.

- [ ] **Step 6: Test delta logic**

Submit the prior pending review (using the printed command). Then push a new commit. Run `/review` again. Expected: only the new commit's diff is in scope; finding count reflects only new code.

- [ ] **Step 7: Test missing artifact (degraded mode)**

In a repo with no `.claude/code-review/best-practices.md`, run `/review <pr>`. Expected: warning printed; review still runs in degraded mode (one default agent flagging obvious bugs); pending review created.

- [ ] **Step 8: Test submit-now flow**

On a small PR, run `/review` and answer `y` at submit prompt, choose `COMMENT`. Verify the review is submitted on GitHub with the summary body intact.

- [ ] **Step 9: Commit Phase 6**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/commands/review.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add /review command"
```

---

## Phase 7: `/review-discussion` command

**Files:**
- Create: `plugins/code-review/commands/review-discussion.md`

**Contract:**

| Aspect | Behavior |
|--------|----------|
| Args | `[pr] <question>`; PR optional, question required |
| Pre-check | None (no eligibility, no delta) |
| Planner (Haiku) | Inputs: question, frontmatter (refs/sections only), PR file list. Output: load plan |
| Targeted load | Only files/hunks/sections/refs from plan |
| Answering agent | Sequential, model `inherit` by default |
| Tools | Read, Grep, WebFetch, WebSearch, Bash (for gh and skill resolution), MCP tools available to the session |
| Output | Prose answer with citations (full-SHA permalinks if grounding in PR code) |
| Optional posting | If actionable issues surfaced: prompt `Draft a pending review? (y/n)` → reuse Phase 6 steps 11–14 |

**Argument parsing:** if `$ARGUMENTS` first token looks like a PR (URL or all-digits number), treat as PR; rest is question. Else: PR auto-detected; whole `$ARGUMENTS` is question. Question is required.

**Planner schema:**
```yaml
diff_scope: full | files: [<paths>] | hunks: ["<path>:<start>-<end>"]
sections_to_load: [<slugs>]
refs_to_load: [<paths or skill:<name>>]
extra_resources:
  skills: [<skill names mentioned in question, not in artifact refs>]
  mcp_tools: [<names>]
  web: <bool>
  files: [<arbitrary paths cited>]
```

### Task 7.1: Author `/review-discussion` command file

- [ ] **Step 1: Write frontmatter**

```yaml
---
allowed-tools: Bash(gh pr-review:*), Bash(gh pr:*), Bash(gh api:*), Bash(git:*), Read, Glob, Grep, WebFetch, WebSearch, Agent
description: Answer a focused question about a PR; lazy-load only what's needed; optionally draft a pending review
disable-model-invocation: false
---
```

- [ ] **Step 2: Author the prompt body**

Required structural elements:

1. **Parse args.** Split `$ARGUMENTS`: if first token is URL or all-digits, PR = first token, question = rest. Else PR = current-branch (`gh pr view --json number`). Question required.
2. **Resolve PR.** Bail with usage if neither explicit nor current-branch.
3. **No eligibility checks. No delta logic.** Proceed regardless of state.
4. **Run lazy-load planner (Haiku via `Agent`).** Pass:
   - The question text.
   - Frontmatter only (parse from `.claude/code-review/best-practices.md`; if missing, pass empty `{}`).
   - `gh pr view <pr> --json files -q '.files[].path'` (file list only, no contents).
   - System prompt instructing the planner to return the planner schema (above) as YAML. The planner picks a tight scope: prefer `files:` over `full`, prefer specific `hunks:` if the question mentions a class/function (planner can grep the file list for matches).
5. **Targeted loads.** Following the plan:
   - Diff: `gh pr diff <pr> -- <files>` (or full diff if `diff_scope: full`). For hunks: load the file then trim to the hunk range.
   - Section bodies: read only the listed sections from the artifact (parse `## <Heading>` boundaries).
   - Refs: for each ref in `refs_to_load`:
     - If starts with `skill:<name>`, resolve in this order: `.claude/skills/<name>/SKILL.md` → `.agents/skills/<name>/SKILL.md` → `~/.claude/skills/<name>/SKILL.md` → `~/.claude/plugins/cache/*/*/skills/<name>/SKILL.md`. Read the first hit.
     - Else treat as file path relative to project root. Read.
     - On miss: warn, continue with empty content.
   - Extra skills (mentioned in the question, not in artifact refs): resolve via the same skill paths.
   - Extra files: read directly with the `Read` tool.
6. **Spawn answering agent.** Use `Agent` with:
   - `subagent_type: general-purpose`
   - **Default: omit the `model` parameter** so it inherits from the parent. If `$ARGUMENTS` contains `--model opus`/`--model sonnet`/`--model haiku`, pass that as `model`.
   - `description:` `Answer PR question`.
   - `prompt:` includes:
     - The question text.
     - All loaded scope (diff, sections, refs, extras).
     - The false-positive baseline, **inlined verbatim**:
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
     - Instruction: answer in prose; cite specific lines/files using the full-SHA permalink format `https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<a>-L<b>` when grounding in PR code; if the answer surfaces concrete actionable issues, list them as a `### Suggested findings` block at the bottom in this YAML schema:
       ```yaml
       suggested_findings:
         - path: <relative path>
           line_start: <int>
           line_end: <int>
           message: <brief, action-oriented>
           evidence: <which loaded resource grounds this>
           confidence: <0-100>
       ```
   - The agent may call `WebFetch`, `WebSearch`, MCP tools, `Read`, `Grep`, `Bash` as needed to ground its answer.
7. **Print the answer.**
8. **Optional posting prompt.** If the agent's response includes a `### Suggested findings` block with at least one finding:
   ```
   I'd suggest <N> inline comments based on this. Draft a pending review? (y/n)
   ```
   On `y`:
   - Get HEAD SHA: `gh pr view <pr> --json headRefOid -q .headRefOid`.
   - Open pending review:
     ```bash
     gh pr-review review --start --pr <n> -R <owner>/<repo> --commit <head-sha>
     ```
     Capture the returned `review-id`.
   - For each suggested finding, add an inline comment:
     ```bash
     gh pr-review review --add-comment \
       --review-id <id> \
       --pr <n> -R <owner>/<repo> \
       --path <p> \
       --line <line_end> \
       --start-line <line_start> \
       --body "$(cat <<EOF
     [discussion · confidence <N>] <message>

     <evidence>
     EOF
     )"
     ```
     Omit `--start-line` if `line_end == line_start`.
   - **Do NOT submit.** Print:
     ```
     Pending review created with <N> inline comments.
     GitHub URL: <pr-url>
     Submit when ready:
       gh pr-review review --submit --review-id <id> --pr <n> -R <owner>/<repo> \
         --event COMMENT|REQUEST_CHANGES|APPROVE \
         --body "<your summary>"

     Submit now? (y/n)
     ```
     On `y`, ask `Event? (COMMENT/REQUEST_CHANGES/APPROVE)` and run the submit command. On `n`, exit cleanly.
9. **No artifact present:** if `.claude/code-review/best-practices.md` doesn't exist, planner runs with empty frontmatter; answering agent runs with no project rules; warn user that artifact-grounded reasoning is unavailable.

### Task 7.2: Smoke-test `/review-discussion`

- [ ] **Step 1: Test on current branch with question only**

```
/review-discussion is the naming on UserService in src/users.ts good?
```
Expected:
- Planner picks `files: [src/users.ts]` and `sections_to_load: [conventions, simplicity]`.
- Answering agent loads only that scope.
- Answer cites specific lines.
- No submit prompt unless agent surfaces actionable issues.

- [ ] **Step 2: Test with explicit PR**

```
/review-discussion 42 does this PR follow OWASP at the auth boundary?
```
Expected: planner picks Security section + relevant files; answer references OWASP and the loaded refs.

- [ ] **Step 3: Test off-artifact skill mention**

```
/review-discussion compare with the riverpod-best-practices skill — does the new state management follow it?
```
Expected: planner adds `riverpod-best-practices` to `extra_resources.skills`; agent has access to that skill's content.

- [ ] **Step 4: Test web access**

```
/review-discussion does this caching strategy align with Redis best practices according to recent docs?
```
Expected: planner sets `web: true`; agent uses WebFetch/WebSearch.

- [ ] **Step 5: Test optional pending-review draft**

```
/review-discussion 42 do you see any null-safety bugs in src/auth.ts? draft a pending review for what you find.
```
Expected: agent surfaces findings; pending review opens; inline comments anchored; no auto-submit.

- [ ] **Step 6: Test missing artifact**

In a repo without `.claude/code-review/best-practices.md`:
```
/review-discussion <pr> any obvious issues?
```
Expected: warning printed about artifact-grounded reasoning unavailable; agent still runs with full PR context.

- [ ] **Step 7: Commit Phase 7**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/commands/review-discussion.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "feat(code-review): add /review-discussion command"
```

---

## Phase 8: README, troubleshooting, repo polish

**Files:**
- Modify: `plugins/code-review/README.md`
- Modify: `README.md` (repo root, optional reference)

### Task 8.1: Author the plugin README

- [ ] **Step 1: Replace `plugins/code-review/README.md` placeholder with full content**

Required sections:

- **Overview** — one paragraph summary.
- **Install**:
  ```bash
  # In a Claude Code session, from a directory containing the agents repo:
  /plugin marketplace add /path/to/wahyubucil/agents
  /plugin install code-review@wahyubucil-agents

  # The CLI dependency:
  gh extension install agynio/gh-pr-review
  ```
- **Commands** — table of all 5 commands with one-line summaries and links to spec sections.
- **Quick start** — three-step flow:
  1. `/init --quick`
  2. `/gather-insight-pr <some past PR>` × N
  3. `/review` on a fresh PR
- **The artifact** — short explanation of `.claude/code-review/best-practices.md`, frontmatter shape, rule format.
- **Configuration** — table of frontmatter fields.
- **Troubleshooting**:
  - `gh pr-review: command not found` → install the extension.
  - Plugin's `gh-pr-review` skill not loaded → `/plugin reload`.
  - Review not submitted → it's pending by design; submit via the printed `gh pr-review review --submit` command or by re-prompting.
  - Bot reviewer comments included in `/gather-insight-pr` → check bot login spelling; add to frontmatter `skip_review_authors`.
- **Versioning the bundled skill** — point to `skills/gh-pr-review/VERSION` and the re-vendoring procedure.
- **Spec link** — to `docs/superpowers/specs/2026-05-05-code-review-plugin-design.md`.

- [ ] **Step 2: Verify Markdown renders**

Run: `head -1 /Users/wahyubucil/projects/github.com/wahyubucil/agents/plugins/code-review/README.md` — expect `# code-review plugin`.

### Task 8.2: Update repo README

- [ ] **Step 1: Add a Plugins section to root `README.md`**

Edit `/Users/wahyubucil/projects/github.com/wahyubucil/agents/README.md` to add (after the `## Contents` section):

```markdown
## Plugins

- **[code-review](./plugins/code-review/)** — project-aware code review with per-section parallel agents and pending GitHub reviews.
```

### Task 8.3: Final smoke pass

- [ ] **Step 1: Reinstall the plugin from a clean state**

```
/plugin uninstall code-review@wahyubucil-agents
/plugin install code-review@wahyubucil-agents
```
Expected: clean reinstall; all 5 commands appear in `/help` or as suggestions.

- [ ] **Step 2: Run end-to-end on a sandbox**

```bash
mkdir -p /tmp/code-review-e2e && cd /tmp/code-review-e2e && git init -q
```
In Claude Code at `/tmp/code-review-e2e`:
- `/init --quick` → file created.
- `/gather-insight-discussion always log structured errors` → rule added.
- (Manually create a PR in a real repo) `/review <pr>` → pending review created, no auto-submit.
- `/review-discussion <pr> any naming issues?` → focused answer.

All commands complete without errors.

- [ ] **Step 3: Commit Phase 8**

```bash
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents add \
  plugins/code-review/README.md \
  README.md
git -C /Users/wahyubucil/projects/github.com/wahyubucil/agents commit -m "docs(code-review): plugin README and root repo reference"
```

---

## Final verification

- [ ] **Plugin marketplace.json** parses, references the plugin at correct source path.
- [ ] **Plugin plugin.json** parses, name is `code-review`.
- [ ] **Vendored skill** at `plugins/code-review/skills/gh-pr-review/{SKILL.md,LICENSE,VERSION}`; auto-loads when plugin is installed.
- [ ] **Template** at `plugins/code-review/templates/best-practices.template.md`; YAML round-trips; 7 sections in expected order.
- [ ] **Five commands** present at `plugins/code-review/commands/{init,gather-insight-discussion,gather-insight-pr,review,review-discussion}.md`; each has YAML frontmatter with `allowed-tools`, `description`, `disable-model-invocation: false`.
- [ ] **Smoke tests** for each command pass (Tasks 3.2, 4.2, 5.2, 6.2, 7.2 outcomes verified).
- [ ] **README** at plugin root and updated repo README.
- [ ] **All commits** present on `main`.

## Risks and mitigations

| Risk | Mitigation |
|------|------------|
| `agynio/gh-pr-review` extension not installed on user's system | First-run detection in commands that use it; print `gh extension install agynio/gh-pr-review`. |
| Vendored skill goes stale | Manual re-vendor; pin tracked in `VERSION`; can be scripted later if friction grows. |
| Section agents fan out too much, slow review | Default sections cap at 7; per-section model lets you assign cheap models where appropriate; can disable a section by removing it from frontmatter. |
| False positives from sections without rules | Section bodies that are empty (only the placeholder comment) → command treats the section as inactive and skips spawning an agent for it. |
| Posting an inline comment to a line not in the delta | Anchor must be a delta-diff line; if the finding cites an older line, mention in body and anchor at the closest delta line. |
| Bot reviewer slips through filter | `skip_review_authors` frontmatter list is user-extendable; built-in list covers common cases. |
| Multi-line finding spans two hunks | The `gh pr-review review --add-comment` command supports `--start-line/--line` across a contiguous range; if non-contiguous, split into two separate inline comments. |
| Plugin command authored without the bundled skill in context | Skill is auto-loaded by Claude Code when the plugin is installed; if absent, command's CLI-flag references still work because each gh invocation is fully spelled out in the prompt. |
