# PRD Template — Comprehensive

Use this template for **standard** and **complex** requests. For trivial/simple requests, use `prd-minimal.md`.

```markdown
# PRD: [Feature Name]

**Author:** Blueprint (AI) + [User Name]
**Date:** [YYYY-MM-DD]
**Status:** Draft
**Version:** 1.0
**Target Repo:** [org/repo]
**Decompose Ready:** Yes

---

## Executive Summary
[2-3 sentences: problem + solution + expected impact]

## Problem Statement
### Current Situation
[What exists today]
### User Impact
[Who is affected, how, severity with evidence]
### Business Impact
[Cost, opportunity cost, strategic importance]
### Why Now
[Urgency driver]

## Goals & Success Metrics
| Goal | Metric | Baseline (source) | Target | Timeframe | Measurement |
|------|--------|--------------------|--------|-----------|-------------|
| ...  | ...    | ...                | ...    | ...       | ...         |

## User Stories
### US-001: [Title]
- **As a** [role]
- **I want to** [action]
- **So that** [benefit]
- **Acceptance Criteria:**
  1. [criterion]
  2. [criterion]
  3. [criterion]
- **Task Hints:** [implementation steps with hour estimates]
- **Dependencies:** [REQ-NNN cross-references]

## Functional Requirements

### P0 — Must Have
#### REQ-001: [Title]
- **Description:** [specific, testable requirement]
- **Acceptance Criteria:**
  1. [criterion]
  2. [criterion]
- **testStrategy:**
  - Acceptance criteria: [what must be true to pass]
  - Verification approach: [how TEST agent validates — automated tests, Playwright, API calls, etc.]
  - Edge cases to test: [specific edge cases]
  - Performance threshold: [if applicable]
- **Technical Spec:** [code examples, API specs]
- **Task Breakdown:** [steps with size estimates]
- **Dependencies:** [other REQ-NNN]

### P1 — Should Have
[same structure]

### P2 — Nice to Have
[same structure]

## Non-Functional Requirements
- **Performance:** [specific thresholds]
- **Security:** [requirements]
- **Scalability:** [expectations]
- **Reliability:** [uptime, error budgets]
- **Accessibility:** [standards]
- **Compatibility:** [browsers, devices, versions]

## Design & UX

### Wireframes
[ASCII or text-based wireframes for each key screen/view. Show layout, component placement, and information hierarchy. One wireframe per user-facing state change.]

Example format:
┌─────────────────────────────────┐
│  Header / Nav                   │
├──────────┬──────────────────────┤
│ Sidebar  │  Main Content Area   │
│          │  ┌────────────────┐  │
│  • Nav 1 │  │  Card / Widget │  │
│  • Nav 2 │  └────────────────┘  │
│          │  [Action Button]     │
└──────────┴──────────────────────┘

### User Flows
[Step-by-step interaction flows for primary user stories. Show: entry point → user action → system response → next state. Reference US-NNN.]

### Interaction Patterns
- **Loading states:** [skeleton, spinner, progressive — be specific]
- **Error states:** [inline validation, toast, error page — per context]
- **Empty states:** [what the user sees before data exists]
- **Responsive behavior:** [breakpoints, layout changes, or "N/A — not user-facing"]

### UI/UX Constraints
[Existing design system, component library, brand guidelines, accessibility standards (WCAG level), or "Greenfield — establish conventions in this feature"]

## Technical Considerations
### System Architecture
[ASCII diagrams encouraged]
### API Specifications
[endpoints, payloads, responses]
### Database Schema Changes
[migrations, new tables/columns]
### Technology Stack
[confirmed from codebase scan]
### External Dependencies
[with failure handling]
### Migration Strategy
[if modifying existing system, include rollback]

## Implementation Roadmap
[Phased, scope-focused — NO timelines, just logical order]

## Dependency Chain
[Logical build order, critical path. **Build dependencies only** — "I can't write/test this without that existing first." Runtime dependencies (calling another component at execution time) are NOT build dependencies and must not appear here. Walk the dependency tree to confirm no circular references before finalizing.]

## Out of Scope
[Explicit boundaries — what we are NOT doing]

## Open Questions & Risks
| # | Question/Risk | Owner | Status |
|---|---------------|-------|--------|
| 1 | [question]    | [who] | Open   |

## Appendix: Task Breakdown Hints
[Summary table for Decompose agent — task name, size estimate, dependencies]
```
