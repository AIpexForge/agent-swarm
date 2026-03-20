# Reviewer Sub-Agent Orchestration

How Blueprint spawns, feeds, and aggregates the 5 review sub-agents.

## Reviewer Roster

| Label | Prompt | Finds | Key Output |
|-------|--------|-------|------------|
| `contradiction` | `agents/reviewers/contradiction-detector/PROMPT.md` | Internal inconsistencies | Contradiction list with both locations |
| `architecture` | `agents/reviewers/architecture-auditor/PROMPT.md` | Bad design, unhandled failures | Failure scenario table |
| `integration` | `agents/reviewers/integration-auditor/PROMPT.md` | Codebase conflicts, duplicated modules | Duplication + pattern conflict report |
| `testability` | `agents/reviewers/testability-auditor/PROMPT.md` | Vague/untestable testStrategies | Draft test snippets + e2e coverage |
| `coherence` | `agents/reviewers/coherence-auditor/PROMPT.md` | Cognitive dissonance — intent vs implementation | Dissonance report with evidence pairs |

All 5 run in **parallel** on `anthropic/claude-sonnet-4-6` with 90s timeout.

## Context Payload

### All reviewers receive:
- Full PRD markdown
- Working plan file (problem statement, scan findings, interview notes, decisions)
- Validation results (score, grade, scope challenge findings)

### Integration Auditor additionally receives (heavy context):
- Directory tree (3+ levels)
- Key module definitions and public interfaces
- Router/API endpoint definitions, schema files
- Existing patterns for similar problems
- Package manifest

### Testability Auditor additionally receives:
- Existing test files listing + sample patterns
- Test framework, CI config, test_cmd

### Architecture Auditor additionally receives:
- Directory tree (top level)
- Stack info from commands.yml

### Contradiction Detector receives:
- PRD + plan file + validation results only (no codebase context)

### Coherence Auditor receives:
- PRD + plan file + validation results only (no codebase context — it reads the plan file for interview/decision context)

## Spawn Pattern

For each reviewer:
1. Read the prompt file from `~/.openclaw/workspace/agent-swarm/agents/reviewers/<type>/PROMPT.md`
2. Assemble the task:
   ```
   <prompt file contents>

   ---

   ## PRD Under Review
   <full PRD markdown>

   ## Working Plan
   <current contents of the working plan file>

   ## Codebase Context
   <appropriate context per reviewer type — see above>

   ## Validation Results
   <15-check score, grade, scope challenge findings>

   Return your review as the structured JSON specified in your prompt.
   ```
3. Spawn: `sessions_spawn(task=<assembled>, label=<label>, model="anthropic/claude-sonnet-4-6", runTimeoutSeconds=90)`

Spawn all 5 in parallel.

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
- Testability Auditor: `requirement_assessments[]`, `e2e_coverage{}`
- Coherence Auditor: `dissonance[]` with `evidence_a`, `evidence_b`, `type`

## Aggregation

### Decision Matrix

| Any Critical? | Action |
|---|---|
| No criticals, all pass | → Proceed to GitHub output |
| No criticals, any concerns | → Auto-incorporate, re-validate, proceed |
| Any critical | → Fix criticals, re-run ONLY the failed reviewer (max 1 retry) |

### Auto-Incorporation Rules

1. **Contradiction Detector:** Resolve contradictions using the reviewer's suggested side
2. **Architecture Auditor:** Add unhandled failure scenarios to Open Questions; update diagram if needed
3. **Integration Auditor:** Update "Existing Code Overlap"; fix phantom references; align patterns
4. **Testability Auditor:** Replace testStrategy with `improved_strategy`; add draft test snippets to appendix
5. **Coherence Auditor:** Realign requirements to match stated intent; flag disproportionate scope to user

After incorporation, re-run validation (15 checks). Do NOT re-run reviewers unless criticals exist.

## Timeout

90 seconds per reviewer. On timeout: treat as pass, note in handoff.
