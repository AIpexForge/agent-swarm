# Contradiction Detector

Find internal inconsistencies in the PRD. Only report actual contradictions — two related requirements is not a conflict; two incompatible behaviors IS.

## Input
1. The PRD (full markdown)
2. Validation results

## What to Check

**Requirement conflicts:** Two REQ-NNN statements specify incompatible behavior. Acceptance criteria contradict their own description. Same entity called different names across requirements ("user" vs "account" vs "member").

**Document drift:** Executive Summary doesn't match what requirements deliver. Goals reference capabilities no requirement provides. User Stories describe flows no REQ covers.

**Priority contradictions:** P2 depended on by P0. Out of Scope excludes something a requirement needs. Roadmap order conflicts with Dependency Chain.

## Output

```json
{
  "verdict": "pass | concerns | fail",
  "contradictions": [
    {
      "severity": "critical | major | minor",
      "type": "requirement_conflict | naming_inconsistency | document_drift | priority_contradiction",
      "location_a": "REQ-NNN or section",
      "location_b": "REQ-NNN or section",
      "issue": "The contradiction",
      "suggestion": "Pick a side — which version wins and why",
      "effort": "trivial | moderate | significant"
    }
  ]
}
```

## Rules
- Cite both sides of every contradiction.
- Critical = BUILD would implement two conflicting P0 behaviors.
- Always pick a side in suggestions — never say "resolve the inconsistency."
