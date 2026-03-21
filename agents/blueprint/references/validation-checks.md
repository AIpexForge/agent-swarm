# PRD Validation Checks

These checks are run by the validation sub-agent spawned in Phase 5. Blueprint does not run these directly — it spawns a sub-agent with the PRD content and this checklist.

## Checks (Comprehensive PRD — 14 items)

| # | Check | Points |
|---|-------|--------|
| 1 | Executive summary exists (20-500 words) | 5 |
| 2 | Problem statement includes user impact | 5 |
| 3 | Problem statement includes business impact | 5 |
| 4 | Goals have SMART metrics | 5 |
| 5 | User stories have acceptance criteria (min 3 each) | 5 |
| 6 | Functional requirements are testable (no vague language) | 5 |
| 7 | Requirements have priority labels (P0/P1/P2) | 5 |
| 8 | Requirements are numbered (REQ-NNN) | 5 |
| 9 | Technical considerations address architecture | 5 |
| 10 | Dependencies are mapped, have no circular references, and distinguish build vs runtime | 5 |
| 11 | Out of scope is defined | 5 |
| 12 | testStrategy present on all P0 requirements | 5 |
| 13 | Task breakdown hints included | 5 |
| 14 | Design & UX section present with wireframes (skip for non-UI features) | 5 |

**Total: 70 points** (65 if non-UI feature — check 14 is N/A)

## Checks (Minimal PRD — 6 items)

For trivial/simple requests using the minimal template, run only these checks:

| # | Check | Points |
|---|-------|--------|
| 1 | Summary exists (10-200 words) | 5 |
| 2 | At least one P0 functional requirement | 5 |
| 3 | All P0 requirements have testStrategy with tool, input, assertion, and failure scenario | 5 |
| 4 | Files affected are listed with real paths | 5 |
| 5 | Out of scope is defined | 5 |
| 6 | Requirements are numbered (REQ-NNN) | 5 |

**Total: 30 points**

## Grading

### Comprehensive PRD
- EXCELLENT: 91%+ (64+/70 or 59+/65)
- GOOD: 83-90% (58-63/70 or 54-58/65)
- ACCEPTABLE: 75-82% (53-57/70 or 49-53/65)
- NEEDS_WORK: <75% (<53/70 or <49/65)

### Minimal PRD
- EXCELLENT: 91%+ (28+/30)
- GOOD: 83-90% (25-27/30)
- ACCEPTABLE: 75-82% (23-24/30)
- NEEDS_WORK: <75% (<23/30)

## Vague Language Detection

Flag these words when used without quantification:
"fast", "easy", "secure", "scalable", "user-friendly", "robust", "efficient", "seamless", "intuitive"

## Sub-Agent Behavior

- Return the score, grade, and list of issues to Blueprint.
- If grade < ACCEPTABLE, Blueprint auto-fixes what it can and re-submits (max 1 retry).
- Remaining issues are flagged to the user.
