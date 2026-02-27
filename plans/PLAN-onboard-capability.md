# Plan: ONBOARD Capability

**Status:** Draft
**Date:** 2026-02-27
**Spec:** specs/FEAT-onboard-capability.md

## What

Add an ONBOARD capability to Blueprint that prepares any GitHub repo for agent-swarm operation. Triggered by user saying "onboard <repo_name>".

## Why

ONBOARD is build order #1 — every other agent depends on repo scaffolding (.agents/commands.yml, AGENTS.md, workflow labels, specs/plans directories) to function. No automated way to produce this scaffolding currently exists.

## Scope

- Deep sampling-based codebase scan (multi-ecosystem)
- Generate .agents/commands.yml with inference comments and formal schema
- Generate AGENTS.md with rich project context
- Create specs/, plans/ directory structure
- Create 6 workflow labels on GitHub
- Handle re-onboarding (diff-aware merge with existing scaffolding)
- Open PR for human review with rollback on failure
- Batch human questions (max 7) for unknowns after max inference effort
- Separate prompt file at agents/onboard/prompt.md

## Not Doing

- Monorepo per-package support (detect + warn only)
- CI/CD setup
- Non-GitHub hosting
- Cron-driven re-onboarding
- Batch multi-repo onboarding
