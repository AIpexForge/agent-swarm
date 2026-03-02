# Deep Trilogy Methodology — Extracted Reference

> Source: [piercelamb/deep-project](https://github.com/piercelamb/deep-project) + [piercelamb/deep-plan](https://github.com/piercelamb/deep-plan)
> License: MIT
> Extracted: 2026-02-27 for Blueprint agent design

---

## Pipeline Overview

```
deep-project (decompose) → deep-plan (plan) → deep-implement (build)
```

For agent-swarm, we map this as:
- **deep-project** → Blueprint interactive mode (interview → decompose → spec)
- **deep-plan** → Blueprint cron mode (research → plan → review → TDD → sections → task issues)
- **deep-implement** → BUILD agent

---

## Phase 1: Decomposition (from deep-project)

### Interview Protocol
- Surface user's mental model through adaptive questioning
- No fixed number of questions — stop when you can propose splits
- Core topics: natural boundaries, ordering/dependencies, uncertainty mapping, existing context
- Technique: open-ended questions, not yes/no; dig deeper on complexity
- **When to stop:** Can propose splits user will recognize, can identify dependencies, can flag parallel work

### Split Heuristics
- Good split: cohesive purpose, bounded complexity, clear interfaces
- Too big: multiple distinct systems, repeated "and also...", >10 implementation sections
- Too small: single function, no architectural decisions, fully specifiable in few sentences
- Single unit is valid — not a failure

### Spec Generation
- Each spec self-contained for planning
- Reference don't duplicate
- Capture interview decisions that shaped the split
- Note dependencies explicitly

### Project Manifest
- Machine-parseable SPLIT_MANIFEST block at top
- Human-readable dependency graph, execution order, cross-cutting concerns below

---

## Phase 2: Deep Planning (from deep-plan)

### Research (Steps 6-7)
1. Extract topics from spec (technologies, patterns, integrations)
2. Ask about codebase research needs
3. Ask about web research topics (derived from spec)
4. Run research subagents in parallel (codebase explore + web search)
5. **Parent combines results** into claude-research.md (subagents don't write files)
6. Always include testing context

### Interview (Step 8)
- Senior architect mindset — accountable for implementation
- Informed by spec + research findings
- 2-4 focused questions per round, open-ended
- Stop when confident to write detailed plan with no assumptions
- Save transcript to claude-interview.md

### Spec Synthesis (Step 10)
- Combine: initial input + research + interview answers
- Output: claude-spec.md (complete synthesized specification)

### Plan Writing (Step 11)
**Critical constraints:**
- Plans are **prose documents**, not code
- Write for unfamiliar reader — fully self-contained
- **Code budget:** ONLY type definitions (fields only), function signatures with docstrings, API contracts, directory structure, config keys
- **NO:** full function bodies, complete tests, import statements, error handling code
- Synthesize all inputs; resolve conflicts with judgment

### External Review (Step 13)
- Send plan to independent reviewer (different model)
- Reviewer checks: footguns, edge cases, security, performance, architecture, ambiguity
- Integration: accept/reject suggestions with documented rationale
- Update plan with accepted feedback

### TDD Approach (Step 16)
- Mirror plan structure with test stubs
- Stubs = prose descriptions or minimal signatures, NOT full test implementations
- Follow project's existing testing conventions (or recommend for new projects)

### Section Splitting (Steps 18-20)
- Section index with SECTION_MANIFEST block + dependency graph
- Each section **completely self-contained** — implementable in isolation
- Tests FIRST in each section
- Parallel execution via batched subagents
- Content: stubs/signatures only, not full implementations

---

## Key Principles to Preserve

1. **Interview → Research → Plan → Review → TDD → Sections** (this sequence matters)
2. **Multi-perspective review** catches blind spots
3. **Plans are blueprints, not buildings** — prose with minimal code stubs
4. **Self-contained sections** — implementer needs only that section file
5. **TDD-first** — tests defined before implementation in every section
6. **Parent orchestrates, subagents return** — no subagent file writes
7. **Adaptive interviews** — no fixed script, stop when understanding is sufficient
