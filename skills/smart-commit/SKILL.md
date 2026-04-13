---
name: smart-commit
description: Smart commit — commit only files modified in the current session, grouped into logical units. TRIGGER when user says "commit", "smart commit", "smart-commit", "commit my changes", "commit session changes", "commit what I changed", "commit this", or invokes /smart-commit. Stages and commits only session-touched files — never unrelated changes.
tools: Bash
---

## Current Git State
- Branch: !`git branch --show-current`
- Status: !`git status --short`

You are a senior developer managing git history.
I have made changes to the codebase and I need you to commit ONLY the files modified in THIS SESSION.

**Context/Hint:** $ARGUMENTS

**Task:**

1. **Analyze Session:** Identify files changed by the current session or explicitly mentioned in the context.
   - Hint: Check the `git status` and compare with the files modified by recent `write`, `edit`, or `bash` tool calls in this session history.
   - **CRITICAL:** Ignore any changes made outside this session (e.g., untracked files from before, or changes made manually in the editor that are unrelated).
2. **Group:** Identify distinct logical units of work (e.g., "fix login bug", "refactor header", "update docs").
3. **Execute:** For *each* logical group found in the SESSION changes:
   a. Stage **only** the relevant files (`git add <file_paths>`).
   b. Create a conventional commit message (e.g., `feat: ...`, `fix: ...`, `chore: ...`).
   c. Commit the changes with signing enabled (`git commit -S`).
4. **Constraints:**
   - **Scope:** ONLY commit files modified by this agent session. Leave everything else unstaged.
   - **Atomicity:** Do NOT dump unrelated changes into a single commit.
   - **Safety:** Do NOT reset, checkout, or discard any changes.
