# Plan: PLAN Phase B — Task Decomposition

**Date:** 2026-03-01
**Spec:** specs/FEAT-plan-task-decomposition.md
**Status:** Draft
**Version:** 1.2

---

## What

Blueprint's cron-driven capability to automatically decompose merged spec PRs into task issues that BUILD can pick up. Bridges the gap between "spec approved" and "pipeline has work."

## Why

Without this, every merged spec requires manual task issue creation — tedious, error-prone, and blocks the entire downstream pipeline (BUILD → TEST → REVIEW → MERGE).

## Key Decisions

- **Blueprint runs it** as a cron capability (not a separate agent)
- **Native decomposition** — LLM-based decomposition directly to `gh issue create` via Decompose agent
- **~30 min BUILD time** per task (agent execution time, not human hours)
- **Sub-agent pattern** — Blueprint spawns a focused decomposition sub-agent per spec
- **Two-pass dependencies** — create all issues first, backfill `depends_on` with real numbers
- **No decomp_id** — duplicate detection uses plan-comment + title matching (deterministic, no hash required)
- **Labels as state** — `plan:draft` (preserved) + `decomposed` (added) drive detection
- **Pre-creation scope review (P0)** — Blueprint reviews the decomposition plan comment BEFORE any issues are created. Classifies tasks as small/medium/large, splits large tasks (>60m) and merges tightly-coupled small ones in the plan, then approves. No post-creation issue mutation.
- **`size` field in metadata** — each task issue carries `size: small|medium|large`; tasks reaching issue creation must be ≤ medium
- **Backfill-only retry** — on retry, if all issues exist with `depends_on: pending`, run Pass 2 only (no re-creation)

## Scope

- **9 P0 requirements:** REQ-001 (find-work.py), REQ-004 (feature branch), REQ-005a (label bootstrap), REQ-003 (task issue structure), REQ-006 (decomposition prompt), REQ-007 (pre-creation scope review), REQ-002 (sub-agent spawning), REQ-008 (idempotency), REQ-005b (plan issue update)
- **2 P1 requirements:** REQ-009 (dependency ordering), REQ-010 (Telegram notification)
- Key artifacts: `find-work.py`, `agents/plan/decompose/AGENTS.md`, cron wrapper

## Dependency Chain (DAG — no cycles)

```
REQ-001 → REQ-004 → REQ-003 → REQ-006 → REQ-007
REQ-001 → REQ-005a
REQ-001 + REQ-004 → REQ-002
REQ-001 + REQ-003 → REQ-008
REQ-002 + REQ-003 + REQ-007 → REQ-005b → REQ-010 (P1)
REQ-003 + REQ-006 → REQ-009 (P1)
```

## Out of Scope

Cross-spec dependencies, parallel decomposition, subtasks, post-creation issue mutation.
