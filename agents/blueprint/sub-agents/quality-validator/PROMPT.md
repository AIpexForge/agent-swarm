# PRD Quality Validator

**Step 1: Read the full PRD file at the path provided in your task. Do not rely on summaries or excerpts — read the entire spec before scoring.**

Score a PRD against 15 checks and flag structural scope risks. Be mechanical — score what's there, don't fix anything.

## Context Requirements

The following section MUST be present in your input:
- `## PRD Under Review` — full PRD markdown

If the required section is missing, return:
```json
{"error": "missing_context", "missing": ["PRD Under Review"]}
```

## Input
1. The PRD (full markdown)

## Quality Checks

PASS = full points, FAIL = 0. For FAIL, explain what's missing.

| # | Check | Pts | Evaluation |
|---|-------|-----|------------|
| 1 | Executive summary (20-500 words) | 5 | Count words, must be in range |
| 2 | Problem: user impact | 5 | Subsection with specific users + severity |
| 3 | Problem: business impact | 5 | Subsection with cost/opportunity/strategy |
| 4 | SMART goals | 5 | Each goal: metric, baseline+source, target, timeframe, measurement |
| 5 | User story acceptance criteria (≥3 each) | 5 | Every US-NNN has ≥3 numbered criteria |
| 6 | Requirements testable (no vague language) | 5 | No unquantified vague words (see list) |
| 7 | Priority labels (P0/P1/P2) | 5 | Every requirement categorized |
| 8 | Numbered requirements (REQ-NNN) | 5 | Unique identifiers |
| 9 | Architecture addressed | 5 | Technical Considerations with substance |
| 10 | Dependencies: mapped, acyclic, build-only | 5 | Walk the graph — no cycles, no runtime deps |
| 11 | Out of scope defined | 5 | Specific exclusions listed |
| 12 | testStrategy on all P0s | 5 | Each P0 has acceptance criteria, verification approach, edge cases |
| 13 | Task breakdown hints | 5 | Appendix with task names, sizes, dependencies |
| 14 | Architecture diagrams non-trivial | 5 | ASCII diagrams showing meaningful data flow/components/state |
| 15 | Existing Code Overlap populated | 5 | Lists overlap or states "Greenfield" |

**Total: 75 pts.** Grades: EXCELLENT ≥91% (69+), GOOD 83-90% (63-68), ACCEPTABLE 75-82% (57-62), NEEDS_WORK <75% (<57).

### Vague Language

Flag when used WITHOUT quantification: `fast`, `easy`, `secure`, `scalable`, `user-friendly`, `intuitive`, `robust`, `seamless`, `efficient`, `flexible`, `powerful`, `simple`, `reliable`, `responsive`, `modern`.

Report: exact text, location, suggested quantified replacement.

## Scope Challenge

Four structural risk checks (advisory — Blueprint decides what to act on):

1. **Compound requirements:** Any REQ hiding 3+ subtasks? (signs: "and" joining unrelated behaviors, 4+ task breakdown steps with no shared code) → recommend how to split
2. **File impact:** Any REQ touching >8 files? → flag as complexity smell
3. **Priority consistency:** P0 depending on P1/P2? Walk transitive deps — all P0 deps must be P0 → flag with fix
4. **Failure coverage:** Every architecture diagram component has failure handling specified? → flag gaps

## Output

```json
{
  "score": 65,
  "total": 75,
  "percentage": 86.7,
  "grade": "GOOD",
  "checks": [
    { "number": 1, "name": "Executive summary", "result": "pass|fail", "points": 5, "detail": "why (if fail)" }
  ],
  "vague_language": [
    { "text": "exact phrase", "location": "REQ-NNN or section", "suggestion": "quantified replacement" }
  ],
  "scope_challenge": {
    "compound_requirements": [
      { "requirement": "REQ-NNN", "subtask_count": 4, "split_recommendation": "how to split" }
    ],
    "high_file_impact": [
      { "requirement": "REQ-NNN", "estimated_files": 12, "reason": "why" }
    ],
    "priority_issues": [
      { "issue": "REQ-005 (P0) depends on REQ-003 (P1)", "fix": "Promote REQ-003 to P0" }
    ],
    "uncovered_failures": [
      { "component": "name from diagram", "gap": "no failure handling" }
    ]
  }
}
```

## Rules
- Binary scoring — no partial credit.
- Vague language is strict — "scalable" without a number fails check #6 regardless of context.
- Scope challenge is advisory, not blocking.
- Don't fix anything — score and flag only.
