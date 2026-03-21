# PRD Reviewers

Blueprint spawns these 4 review sub-agents in parallel after validation passes (≥ ACCEPTABLE). Each runs in a fresh session. Skip for trivial/simple requests.

## Reviewer 1: Architecture Review
- Is the proposed architecture sound?
- Scalability concerns not addressed?
- External dependency failure modes and fallbacks?
- Migration strategy covers rollback?

## Reviewer 2: Requirements Completeness
- Gaps in functional requirements?
- Acceptance criteria cover edge cases?
- Error states and failure modes defined?
- Dependency chain logically sound?

## Reviewer 3: Scope & Feasibility
- Scope realistic for the described system?
- Hidden complexities not captured?
- Out-of-scope section adequate?
- Task breakdown estimates reasonable?

## Reviewer 4: testStrategy Review
- Every P0 requirement's testStrategy actually verifiable?
- Verification approaches concrete enough for autonomous TEST agent?
- Requirements that would be hard to test automatically?
- Performance thresholds measurable?

## Output Contract

Each reviewer returns:
```json
{
  "verdict": "pass | concerns | fail",
  "issues": [
    {
      "severity": "critical | major | minor",
      "section": "REQ-003",
      "issue": "Missing error handling for rate limit exceeded",
      "suggestion": "Add acceptance criterion for 429 response handling"
    }
  ]
}
```

## Decision Logic (Blueprint's responsibility)

- All pass → proceed to GitHub output
- Any concerns (no criticals) → Blueprint auto-incorporates suggestions, re-validates, proceeds
- Any critical → Blueprint fixes critical issues, re-runs that specific reviewer (max 1 retry), then proceeds or escalates to user
