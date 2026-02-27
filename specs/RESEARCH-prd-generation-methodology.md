# Research: PRD Generation Methodology for PLAN Agent

> **Date:** 2026-02-27  
> **Status:** Complete  
> **Author:** Servo (AI) + George Sapp  
> **Outcome:** Adapted prd-taskmaster methodology selected

---

## Problem

Taskmaster's front half (human writes PRD alone, one-shot `parse-prd`) is a non-starter for autonomous agent workflows. We need an AI-assisted interview → PRD pipeline that outputs documents Taskmaster's `parse-prd` can consume.

## Candidates Evaluated

### 1. Deep Trilogy `/deep-project` (Pierce Lamb)
- **What:** Interview → decompose into scoped "splits" → spec per split → `/deep-plan`
- **Compatibility:** Low-medium. Outputs spec files per split, not a single PRD. Needs translation layer.
- **Interview quality:** Strong — structured Q&A, dependency mapping between splits.
- **Issue:** Tightly coupled to Claude Code plugin system. Not easily extracted standalone.
- **Verdict:** Good interview protocol, wrong output format.

### 2. Intent Blueprint (`@intentsolutions/blueprint`)
- **What:** 17-question structured intake → up to 22 docs including PRD, architecture, user stories, task generation.
- **Has interview mode:** `blueprint interview` — AI-guided intake session.
- **Compatibility:** Medium-high. Generates a PRD directly. Could use just interview → PRD portion.
- **Issue:** Enterprise-heavy, beta, own task generation overlaps/conflicts with Taskmaster.
- **Verdict:** Interesting but overkill, and its task generation competes with Taskmaster.

### 3. BMAD Method (v6)
- **What:** Full agile lifecycle with 12+ agent personas, scale-adaptive.
- **Compatibility:** Low. Complete competing methodology.
- **Verdict:** Rejected — overkill. Previously evaluated and rejected.

### 4. PRISM Core (`nilukush/prism-core`)
- **What:** Full product management platform (FastAPI + Next.js + Postgres + Redis + Qdrant).
- **Compatibility:** N/A — it's a SaaS platform, not a methodology.
- **Verdict:** Rejected — wrong category entirely.

### 5. prd-taskmaster (`anombyte93/prd-taskmaster`) ✅ SELECTED
- **What:** Claude Code skill — 12-step interview → comprehensive PRD → Taskmaster integration.
- **Interview:** 13 questions across 4 categories (Essential, Technical, Taskmaster-specific, Open-ended).
- **Output:** PRD with 12 sections, directly targets `.taskmaster/docs/prd.md`.
- **Validation:** 13 automated quality checks with letter grading.
- **Compatibility:** Purpose-built for Taskmaster's `parse-prd`. Direct fit.
- **Issue:** Claude Code skill format — needs adaptation for OpenClaw/Telegram.
- **Verdict:** Best fit. Extract the methodology (interview protocol, PRD template, validation), drop the Claude Code tooling.

### 6. Taskmaster's Own Template
- **What:** Built-in PRD template at `.taskmaster/templates/example_prd.txt`.
- **Structure:** `<context>` (Overview, Core Features, UX) + `<PRD>` (Tech Architecture, Roadmap, Dependency Chain, Risks, Appendix).
- **Issue:** Thin — no interview, no validation, no testStrategy.
- **Verdict:** Useful as a reference for what `parse-prd` expects, but insufficient alone.

## Decision

**Adapt prd-taskmaster's methodology** for the PLAN agent:

**Keep:**
- 13-question interview protocol (batched in 3 rounds for Telegram)
- Comprehensive PRD template (12 sections)
- 13 automated quality validation checks
- Vague language detection

**Add:**
- `testStrategy` field per requirement (for TEST agent)
- Pre-interview codebase scan (reduce unnecessary questions)
- 4 fresh-session sub-agent review step
- PR review comment handling (cron-driven)

**Drop:**
- Claude Code skill format / `AskUserQuestion` tool
- CLAUDE.md / codex.md generation
- Execution modes (BUILD agent handles execution)
- `.taskmaster/` local directory scaffolding (GitHub is our state layer)
- Tracking scripts (time tracking, rollback, etc.)

**Runtime dependency:** `task-master-ai` npm package for `parse-prd`, `expand-all`, `analyze-complexity` CLI commands.

## References

- prd-taskmaster: https://github.com/anombyte93/prd-taskmaster
- Deep Trilogy blog: https://pierce-lamb.medium.com/the-deep-trilogy-claude-code-plugins-for-writing-good-software-fast-33b76f2a022d
- Taskmaster: https://github.com/eyaltoledano/claude-task-master
- Intent Blueprint: https://github.com/intent-solutions-io/intent-blueprint-docs
