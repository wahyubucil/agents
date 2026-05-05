# code-review plugin

## Overview

`code-review` is a Claude Code plugin for project-aware code reviews. Instead of generic
"is this good code?" feedback, it drives every review from a committed best-practices
artifact (`.claude/code-review/best-practices.md`) that lives in your repo, so the rules
travel with the project and grow as your team learns. The plugin ships five commands:
`/init` scaffolds the artifact, `/gather-insight-discussion` and `/gather-insight-pr`
build it up from real conversations and past human reviews, `/review` runs a full
project-aware review with parallel section agents and opens a pending GitHub review,
and `/review-discussion` answers focused questions about a PR with lazy-loaded scope.

## Install

```bash
# In a Claude Code session, with an absolute path to a local clone of the agents repo:
/plugin marketplace add /path/to/wahyubucil/agents
/plugin install code-review@wahyubucil-agents

# The CLI dependency:
gh extension install agynio/gh-pr-review
```

The plugin bundles a `gh-pr-review` skill that auto-loads with the plugin and teaches
Claude how to drive the CLI. The `gh` extension itself is a separate install — the
skill alone won't work without the binary on your PATH.

## Commands

| Command | Purpose |
|---------|---------|
| `/init [--quick]` | Scaffold `.claude/code-review/best-practices.md` for the current project. |
| `/gather-insight-discussion [topic]` | Distill a single rule from a focused dialogue and append it to the artifact. |
| `/gather-insight-pr <pr>` | Mine generalizable rules from human reviewer comments on a past PR. |
| `/review [pr]` | Run a full project-aware code review with parallel section agents; opens a pending GitHub review. |
| `/review-discussion [pr] <question>` | Answer a focused question about a PR with lazy-loaded scope; optionally roll into a pending review. |

## Quick start

1. `/init --quick` — scaffold the artifact with sensible defaults.
2. `/gather-insight-pr <some-past-pr>` × N — feed in past PRs where humans left thoughtful
   review comments. Each run distills generalizable rules from those comments and
   appends them to the artifact. Repeat for half a dozen PRs to bootstrap a useful
   ruleset.
3. `/review` on a fresh PR — run a project-aware review using the rules you just built.

## The artifact (`best-practices.md`)

The artifact is the source of truth for what your project considers good code. It lives
at `.claude/code-review/best-practices.md` in each project and is committed to git so
the rules travel with the codebase.

It has two parts:

- **YAML frontmatter** (machine-readable): defines `sections`, `min_confidence`, and
  `skip_authors`. Section keys here are slugs — lowercase with underscores instead of
  spaces.
- **Markdown body** (human-readable): one heading per section, with rules nested under
  each heading. Body headings are Title Case.

Frontmatter section keys round-trip with body headings via lowercase + spaces→underscores
(e.g. frontmatter `error_handling` ↔ body heading `## Error Handling`).

Each rule follows this shape:

```markdown
- **Always check for null** before accessing nested properties on API responses
  - **Why:** API returns sparse objects when fields are unauthorized
  - **Example:** `user?.profile?.name ?? 'Anonymous'`
```

The headline imperative is the rule itself; **Why** records the rationale (so future
readers can decide whether the rule still applies); **Example** is a concrete snippet.

## Configuration

Frontmatter fields:

| Field | Type | Description |
|-------|------|-------------|
| `sections` | map | Section name → `{ refs, model }`. Frontmatter key is the slug (lowercase, underscores); body heading is Title Case. |
| `sections.<x>.refs` | string[] | File paths or `skill:<name>`. |
| `sections.<x>.model` | string | `inherit`/`sonnet`/`opus`/`haiku`. Default `inherit`. |
| `min_confidence` | int in [0, 100] | Default 80. Findings below this are dropped. |
| `skip_authors` | string[] | PR authors whose PRs `/review` skips wholesale. |
| `skip_review_authors` | string[] | (Optional) Reviewer logins `/gather-insight-pr` ignores. |

## Troubleshooting

- **`Missing dependency: gh extension install agynio/gh-pr-review`** (or the gh CLI shows
  `unknown command "pr-review"`) — run `gh extension install agynio/gh-pr-review` to
  install. The bundled skill teaches Claude how to use the binary, but doesn't ship the
  binary itself.
- **Plugin's `gh-pr-review` skill not loaded** — run `/plugin reload`, or restart Claude
  Code. The skill is bundled with the plugin and should auto-load on install.
- **Review not submitted** — pending reviews are by design; submit via the printed
  `gh pr-review review --submit ...` command, or via the GitHub UI. Note: re-running
  `/review` only detects prior **submitted** self-reviews via the delta logic. Pending
  (unsubmitted) reviews aren't auto-detected — submit or discard the existing pending
  review on GitHub before re-running, otherwise you may end up with multiple stacked
  pending reviews on the same PR.
- **Bot reviewer comments included in `/gather-insight-pr`** — the command filters by
  reviewer login. Check the bot login spelling and add it to frontmatter
  `skip_review_authors`.
- **Wrong section assigned to a rule from `/gather-insight-discussion`** — use the
  `Override?` prompt the command offers, or edit the artifact directly afterward.
- **Artifact format errors** — re-run `/init` and answer `r` (replace) at the existence
  prompt, OR use `/init --quick` to skip prompts. Both rewrite the artifact from the
  template (you'll lose all rules — back up first if needed).

## Versioning the bundled skill

The plugin bundles a snapshot of the upstream `gh-pr-review` skill. Its version is
pinned in `skills/gh-pr-review/VERSION` (commit SHA + date).

To re-vendor against a newer upstream commit, follow the fetch commands in
`docs/superpowers/plans/2026-05-05-code-review-plugin.md` Phase 1 with the new commit
SHA, then commit. Bump `skills/gh-pr-review/VERSION` to match.

## Spec

Full design context: `docs/superpowers/specs/2026-05-05-code-review-plugin-design.md`.
