# Plan: Agent Workflow System

> **Status:** Draft  
> **Date:** 2026-02-27  
> **Spec:** [specs/FEAT-agent-workflow-system.md](../specs/FEAT-agent-workflow-system.md)

## What

A collection of independent AI agents that automate the software development lifecycle — planning, building, testing, reviewing, and merging — using GitHub as the shared coordination layer.

## Why

The monolithic orchestrator approach (agentic-workflow) couples all lifecycle stages into a single script. This creates brittleness: one failure stalls everything, and the system is hard to extend. Decoupled agents with clear contracts are more resilient, testable, and composable.

## Goals

1. Each agent operates independently on its own cron schedule
2. GitHub labels, issues, and PRs are the only coordination mechanism
3. Any AI agent framework can operate on the conventions (agent-agnostic)
4. Human stays in the loop at spec approval and escalation points
5. System works across any repo in the AIpexForge organization

## Non-Goals (v1)

- Parallel task execution (sequential only)
- Model optimization (all agents on Opus)
- Cross-repo learning
- Self-hosted CI (uses GitHub Actions)

## Success Criteria

- Agents can take a human-approved spec and autonomously build, test, review, and merge the implementation
- Retry + escalation works reliably
- First target repo (snaphappy) successfully onboarded and processed
