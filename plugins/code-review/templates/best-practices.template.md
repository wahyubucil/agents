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
