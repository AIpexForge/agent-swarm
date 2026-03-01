# Plan: PLAN Phase B — Task Decomposition

**Date:** 2026-03-01
**Spec:** specs/FEAT-plan-task-decomposition.md
**Status:** Draft

---

## What

Blueprint's cron-driven capability to automatically decompose merged spec PRs into task issues that BUILD can pick up. Bridges the gap between "spec approved" and "pipeline has work."

## Why

Without this, every merged spec requires manual task issue creation — tedious, error-prone, and blocks the entire downstream pipeline (BUILD → TEST → REVIEW → MERGE).

## Key Decisions

- **Blueprint runs it** as a cron capability (not a separate agent)
- **No Taskmaster** — native LLM decomposition directly to `gh issue create`
- **~30 min BUILD time** per task (agent execution time, not human hours)
- **Sub-agent pattern** — Blueprint spawns a focused decomposition sub-agent per spec
- **Two-pass dependencies** — create all issues first, backfill `depends_on` with real numbers
- **Hash-based idempotency** — `decomp_id` in metadata prevents duplicates
- **Labels as state** — `plan:draft` (preserved) + `decomposed` (added) drive detection
- **Scope review deferred to v2** — trust prompt sizing, BUILD escalates if tasks run long

## Scope

- 8 P0 requirements, 2 P1
- ~18h estimated build effort
- Key artifacts: `find-work.py`, `agents/plan/decompose/AGENTS.md`, cron wrapper

## Out of Scope

Cross-spec dependencies, parallel decomposition, Taskmaster, subtasks, scope review (v2).
