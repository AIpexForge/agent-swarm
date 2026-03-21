# PRD Template — Minimal

Use this template for **trivial** and **simple** requests. For standard/complex, use `prd-template.md`.

```markdown
# PRD: [Feature Name]

**Author:** Blueprint (AI) + [User Name]
**Date:** [YYYY-MM-DD]
**Status:** Draft
**Version:** 1.0
**Target Repo:** [org/repo]
**Classification:** [Trivial | Simple]

---

## Summary
[1-2 sentences: what we're changing and why]

## Functional Requirements

### P0 — Must Have
#### REQ-001: [Title]
- **Description:** [specific, testable requirement]
- **Acceptance Criteria:**
  1. [criterion]
  2. [criterion]
- **testStrategy:**
  - Tool: [curl, Playwright, bun test, etc.]
  - Input: [concrete test data]
  - Assertion: [exact expected result]
  - Failure scenario: [at least one]
- **Dependencies:** [other REQ-NNN or "None"]

[Add more REQs as needed — typically 1-3 for trivial/simple]

## Technical Considerations
- **Files affected:** [list specific file paths from codebase scan]
- **Approach:** [brief implementation approach — 2-3 sentences]
- **Risks:** [anything non-obvious, or "None identified"]

## Out of Scope
[1-2 bullet points — explicit boundaries]

## Open Questions
[Any unresolved items, or "None — scope is clear"]
```

## When to Use

- Trivial: Single endpoint, config change, <10 lines changed
- Simple: 1-2 components, clear scope, <1 day of work
- If you discover the scope is larger than expected during research, upgrade to the comprehensive template
