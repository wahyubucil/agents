---
name: code-simplifier
description: Simplify and refine recently modified code for clarity, consistency, and maintainability without changing functionality. Use after writing or modifying code to improve readability and reduce complexity. Trigger when the user says "simplify", "simplify code", "clean up code", "refine code", "review for simplicity", or after completing a coding task that could benefit from a simplification pass.
---

Review changes in the current branch, or in the scope the user specifies. Apply these criteria without changing behavior:

## 1. Preserve Functionality

Never change what the code does — only how it expresses it. All original features, outputs, and behaviors must remain intact. If a simplification risks altering behavior, skip it.

## 2. Names

Shorten verbose names while keeping them clear. Prefer human-readable concepts (`baseline`) over compound phrases that feel mechanical or academic (`lastObservedDiskContent`). Names should communicate intent at a glance.

## 3. Combine Related Concepts

If two types, functions, or constants overlap significantly, merge them. The fewer distinct concepts a reader must hold in their head, the better.

*Example: two union types sharing 3 of 4 values can become one type with shorter value names.*

## 4. Derivability

If a value can be computed from other values already in scope, don't pass or store it separately. Removing derivable state often simplifies signatures, types, and control flow in one move.

*Example: an `isDirty` parameter that's always `editorContent !== baseline` can be dropped and computed where needed.*

## 5. Reduce Complexity

- Flatten unnecessary nesting and indirection
- Eliminate redundant code and dead abstractions
- Remove comments that restate what the code already says clearly
- Consolidate scattered logic that belongs together
- Prefer straightforward control flow over clever tricks (e.g., if/else or switch over deeply nested ternaries)

## 6. Maintain Balance

Avoid over-simplification that trades readability for brevity. Don't:

- Create overly clever solutions that are hard to understand
- Combine too many concerns into a single function
- Remove helpful abstractions that improve organization
- Prioritize "fewer lines" over readability
- Make the code harder to debug or extend

Clarity beats brevity. Explicit code is often better than compact code.

## 7. Follow Project Standards

Respect the project's established conventions (from README.md, CLAUDE.md, AGENTS.md, linter configs, existing patterns). Check the project's docs directory if one exists for additional context on architecture, decisions, or style guides. When the project has a clear style, match it rather than imposing external conventions.

## Process

1. Identify the recently modified code (check git diff or the current session's changes)
2. Analyze each change for simplification opportunities using the criteria above
3. Apply refinements, ensuring no functionality changes
4. Verify the result is simpler and at least as readable as before

## Scope

Only touch code in the specified scope — recently modified files by default. Do not refactor unrelated code unless explicitly asked.
