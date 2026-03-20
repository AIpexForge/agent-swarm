# Reviewer Sub-Agent Orchestration

How Blueprint spawns, feeds, and aggregates the 4 review sub-agents.

## Reviewer Roster

| Label | Prompt | Finds | Key Output |
|-------|--------|-------|------------|
| `contradiction` | `agents/reviewers/contradiction-detector/PROMPT.md` | Internal inconsistencies | Contradiction list with both locations |
| `architecture` | `agents/reviewers/architecture-auditor/PROMPT.md` | Bad design, unhandled failures | Failure scenario table |
| `integration` | `agents/reviewers/integration-auditor/PROMPT.md` | Codebase conflicts, duplicated modules | Duplication + pattern conflict report |
| `testability` | `agents/reviewers/testability-auditor/PROMPT.md` | Vague/untestable testStrategies | Draft test snippets or failure explanations |

All 4 run in **parallel** on `anthropic/claude-sonnet-4-6` with 90s timeout.

## Context Payload

### All reviewers receive:
- Full PRD markdown
- Validation results (score, grade, scope challenge findings)

### Integration Auditor additionally receives (heavy context):
- Directory tree (3+ levels)
- Key module definitions and public interfaces
- Router/API endpoint definitions
- Schema files
- Existing patterns for similar problems
- Package manifest (package.json / go.mod / etc.)

### Testability Auditor additionally receives:
- Existing test files listing + sample test patterns
- Test framework identification
- CI configuration
- `.agents/commands.yml` test_cmd

### Architecture Auditor additionally receives:
- Directory tree (top level)
- `.agents/commands.yml` stack info

### Contradiction Detector receives:
- PRD only + validation results (no codebase context needed)

## Spawn Pattern

For each reviewer:
1. Read the prompt file from `~/.openclaw/workspace/agent-swarm/agents/reviewers/<type>/PROMPT.md`
2. Assemble the task:
   ```
   <prompt file contents>

   ---

   ## PRD Under Review
   <full PRD markdown>

   ## Codebase Context
   <appropriate context per reviewer type — see above>

   ## Validation Results
   <15-check score, grade, scope challenge findings>

   Return your review as the structured JSON specified in your prompt.
   ```
3. Spawn: `sessions_spawn(task=<assembled>, label=<label>, model="anthropic/claude-sonnet-4-6", runTimeoutSeconds=90)`

Spawn all 4 in parallel.

## Normalized Issue Schema

All reviewers use this base schema for issues:
```json
{
  "severity": "critical | major | minor",
  "section": "REQ-NNN or section name",
  "issue": "Clear description",
  "suggestion": "Concrete fix",
  "effort": "trivial | moderate | significant"
}
```

Each reviewer adds typed extensions:
- Contradiction Detector: `contradictions[]` with `location_a`, `location_b`, `type`
- Architecture Auditor: `failure_scenarios[]`, `design_evaluation{}`
- Integration Auditor: `duplication_check[]`, `pattern_conflicts[]`, `phantom_references[]`
- Testability Auditor: `requirement_assessments[]` with `draft_test`, `improved_strategy`

## Aggregation

### Decision Matrix

| Any Critical? | Action |
|---|---|
| No criticals, all pass | → Proceed to GitHub output |
| No criticals, any concerns | → Auto-incorporate, re-validate, proceed |
| Any critical | → Fix criticals, re-run ONLY the failed reviewer (max 1 retry) |

### Auto-Incorporation Rules

When incorporating reviewer feedback:
1. **Contradiction Detector:** Resolve contradictions using the reviewer's suggested side
2. **Architecture Auditor:** Add unhandled failure scenarios to Open Questions; update architecture diagram if design issues found
3. **Integration Auditor:** Update "Existing Code Overlap" section; fix phantom references; align patterns
4. **Testability Auditor:** Replace testStrategy text with `improved_strategy` from the auditor; add `draft_test` snippets to an appendix

After incorporation, re-run validation (15 checks). Do NOT re-run reviewers — one pass is enough unless criticals exist.

## Timeout

Each reviewer gets 90 seconds. On timeout:
- Log the timeout
- Treat as `pass`
- Note in handoff: "[Reviewer] timed out — manual review recommended"
