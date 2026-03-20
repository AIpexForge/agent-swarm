# Reviewer Sub-Agent Orchestration

How Blueprint spawns, feeds, and aggregates the 4 review sub-agents.

---

## Spawning

Blueprint spawns all 4 reviewers in **parallel** as one-shot sub-agents. Each reviewer runs in a fresh session with no memory of prior interactions.

### Context Payload

Each reviewer receives the same context bundle:

```
1. PROMPT.md — the reviewer's specific prompt (from agents/reviewers/<type>/PROMPT.md)
2. The full PRD markdown
3. Codebase context:
   - Directory tree (top 3 levels)
   - .agents/commands.yml content
   - AGENTS.md content (if exists)
   - package.json / Cargo.toml / go.mod (whatever applies)
   - Existing test files listing (for test-strategy reviewer)
   - Key architectural files (router definitions, schema files, config)
4. Validation results — the 13-check score + any flagged issues
```

### Spawn Pattern

```
Blueprint sends to each reviewer:

"You are [Reviewer Name]. Review the following PRD against the codebase context provided.

## Your Review Prompt
<contents of PROMPT.md>

## PRD
<full PRD markdown>

## Codebase Context
<directory tree, config files, existing patterns>

## Validation Results
<13-check score, flagged issues>

Return your review as the structured JSON specified in your prompt."
```

---

## Aggregation

Blueprint collects all 4 reviewer responses and aggregates:

### Decision Matrix

| Architecture | Requirements | Scope | Test Strategy | Action |
|---|---|---|---|---|
| pass | pass | pass | pass | → Proceed to GitHub output |
| concerns | any | any | any | → Auto-incorporate, re-validate, proceed |
| any | concerns | any | any | → Auto-incorporate, re-validate, proceed |
| any | any | concerns | any | → Auto-incorporate, re-validate, proceed |
| any | any | any | concerns | → Auto-incorporate (use `improved_strategies`), re-validate, proceed |
| fail (any) | - | - | - | → Fix criticals, re-run failed reviewer (max 1 retry) |

### Auto-Incorporation Rules

When a reviewer returns `concerns`:
1. For each `major` issue with a `suggestion`: apply the suggestion to the PRD
2. For `improved_strategies` from test-strategy reviewer: replace existing testStrategy text
3. For `missing_requirements` from requirements reviewer: add as P1 unless clearly P0
4. For `scope_adjustments` from scope reviewer: apply priority changes, notify user in handoff
5. Re-run validation (13 checks) after all incorporations
6. Do NOT re-run the reviewers — one pass is enough unless there were criticals

### Failure Handling

When a reviewer returns `fail`:
1. Extract all `critical` severity issues
2. Fix each critical issue in the PRD (Blueprint does this, not a sub-agent)
3. Re-run ONLY the reviewer that failed (max 1 retry)
4. If still fails after retry: proceed anyway, but flag all unresolved criticals in the handoff message and in the PR description

### Handoff Summary

The aggregated review results feed into Blueprint's handoff message:

```
Reviewer verdicts:
• Architecture: pass ✅
• Requirements: concerns → 2 suggestions incorporated ✅
• Scope: pass ✅
• Test Strategy: concerns → 3 testStrategies improved ✅
```

---

## Timeout

Each reviewer gets a **90-second timeout**. If a reviewer doesn't respond:
- Log the timeout
- Treat as `pass` (don't block the pipeline on a hung sub-agent)
- Note in handoff: "Architecture review timed out — manual review recommended"

---

## Cost Efficiency

All 4 reviewers use the same model as Blueprint. They receive a focused payload (PRD + relevant context), not the entire conversation history. Expected token usage per reviewer: ~2-4k input, ~1-2k output.

Total review phase cost: ~4x a single sub-agent call. Worth it for the quality gate.
