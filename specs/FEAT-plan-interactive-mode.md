# Spec: PLAN Agent — Interactive Mode (PRD Generation)

> **Plan:** PLAN agent interactive mode for PRD generation  
> **Status:** Draft — awaiting human approval  
> **Author:** Servo (AI) + George Sapp  
> **Date:** 2026-02-27  
> **Depends on:** FEAT-agent-workflow-system

---

## Overview

PLAN agent's interactive mode conducts a structured interview with the user via Telegram, generates a comprehensive PRD optimized for Taskmaster's `parse-prd` pipeline, validates the PRD quality, and outputs a spec PR + plan issue on GitHub. After the user merges the PR, PLAN's cron mode decomposes the PRD into task issues.

**Key design decisions:**
- Interview methodology adapted from [prd-taskmaster](https://github.com/anombyte93/prd-taskmaster) (12+ question structured interview)
- PRD template targets Taskmaster's `parse-prd` input format
- 13 automated quality checks validate the PRD before submission
- 4 fresh-session sub-agents review the PRD before it hits GitHub
- Taskmaster CLI (`task-master-ai` npm package) is a runtime dependency
- PLAN detects and addresses PR review comments from the user on cron

---

## Invocation

**Trigger:** User messages Servo requesting planning for a feature/repo.

**Flow:**
```
George → Servo (Telegram)
  "I want to plan [feature] for [repo]"
       │
       ▼
Servo spawns PLAN as a sub-agent, passing:
  - target_repo: AIpexForge/<repo>
  - feature_description: <initial description from George>
  - channel: telegram
```

PLAN runs as a one-shot sub-agent session. It exits after creating the PR.

---

## Phase 1: Discovery Interview

PLAN conducts a conversational interview over Telegram. Questions are batched in rounds to keep the conversation natural — not dumped all at once.

### Pre-Interview: Codebase Scan

Before asking questions, PLAN:
1. Clones the target repo
2. Reads `.agents/commands.yml` for stack info
3. Scans the codebase structure
4. Pre-fills answers it can infer (tech stack, existing patterns, integration points)

This reduces the interview to only what PLAN can't determine from code.

### Round 1 — Problem & Users

1. **What problem are we solving?** (user pain point, business impact, severity)
2. **What's the proposed solution at a high level?**
3. **Any hard constraints?** (technical, timeline, resources, compliance)

*Wait for response. Follow up if answers are vague — state assumptions and ask for confirmation rather than re-asking the same question.*

### Round 2 — Technical Context

4. **Existing codebase or greenfield? What's the stack?** *(pre-filled from scan — confirm or correct)*
5. **What does this integrate with?** (APIs, services, other repos)
6. **Performance/scale expectations?**

*Only ask questions PLAN couldn't answer from the codebase scan.*

### Round 3 — Success & Scope

7. **How do we measure success?** (push for quantifiable metrics: baseline → target)
8. **What's explicitly out of scope?**
9. **Edge cases, risks, anything else?**

### Smart Defaults

If the user gives terse answers:
- PLAN states its assumptions explicitly
- Asks for confirmation: "I'm assuming X based on the codebase — correct?"
- Documents all assumptions in the PRD's Open Questions section

---

## Phase 2: Research (Optional, Autonomous)

If the feature touches unfamiliar tech or external APIs:
1. PLAN spawns research sub-agents (parallel) to investigate
2. Results feed into the PRD's Technical Considerations section
3. User is NOT interrupted for this phase

**Trigger conditions:**
- Feature references an external API PLAN hasn't seen in the codebase
- User mentions a technology not in `.agents/commands.yml`
- User explicitly requests research ("look into how X works")

---

## Phase 3: PRD Generation

PLAN generates the PRD using the comprehensive template structure.

### PRD Sections

1. **Executive Summary** — 2-3 sentences: problem + solution + expected impact
2. **Problem Statement**
   - Current Situation
   - User Impact (who, how, severity with evidence)
   - Business Impact (cost, opportunity cost, strategic importance)
   - Why Now
3. **Goals & Success Metrics** — SMART format per goal:
   - Metric, Baseline (with source), Target, Timeframe, Measurement Method
4. **User Stories** — Each with:
   - As a / I want to / So that
   - Acceptance criteria (minimum 3 per story)
   - Task breakdown hints with hour estimates
   - Dependencies (REQ-NNN cross-references)
5. **Functional Requirements** — Prioritized, numbered:
   - **P0 (Must Have):** Critical for launch
   - **P1 (Should Have):** Important but not blocking
   - **P2 (Nice to Have):** Future enhancement
   - Each requirement includes:
     - Description
     - Acceptance criteria
     - **testStrategy** (acceptance criteria + verification approach — what TEST agent validates against)
     - Technical specification (code examples, API specs)
     - Task breakdown with size estimates
     - Dependencies
6. **Non-Functional Requirements** — Performance, security, scalability, reliability, accessibility, compatibility
7. **Technical Considerations**
   - System architecture (with ASCII diagrams)
   - API specifications
   - Database schema changes
   - Technology stack
   - External dependencies + failure handling
   - Migration strategy (if existing system)
8. **Implementation Roadmap** — Phased, scope-focused (no timelines)
9. **Dependency Chain** — Logical build order, critical path
10. **Out of Scope** — Explicit boundaries
11. **Open Questions & Risks** — With owners and deadlines
12. **Appendix: Task Breakdown Hints** — Summary for Taskmaster consumption

### testStrategy Field

Every functional requirement includes a `testStrategy` section:

```markdown
**testStrategy:**
- **Acceptance criteria:** [What must be true for this to pass]
- **Verification approach:** [How TEST agent validates — automated tests, manual Playwright testing, API calls, etc.]
- **Edge cases to test:** [Specific edge cases]
- **Performance threshold:** [If applicable — response time, throughput]
```

This is what the TEST agent validates against independently — it never sees BUILD's implementation approach, only the spec's testStrategy.

---

## Phase 4: Validation

### Automated Quality Checks (13 Checks)

Run against the generated PRD before review:

| # | Check | Category | Points |
|---|-------|----------|--------|
| 1 | Executive summary exists (20-500 words) | Required | 5 |
| 2 | Problem statement includes user impact | Required | 5 |
| 3 | Problem statement includes business impact | Required | 5 |
| 4 | Goals have SMART metrics | Required | 5 |
| 5 | User stories have acceptance criteria (min 3 each) | Required | 5 |
| 6 | Functional requirements are testable (no vague language) | Required | 5 |
| 7 | Requirements have priority labels (P0/P1/P2) | Required | 5 |
| 8 | Requirements are numbered (REQ-NNN) | Required | 5 |
| 9 | Technical considerations address architecture | Required | 5 |
| 10 | Dependencies are mapped | Completeness | 5 |
| 11 | Out of scope is defined | Completeness | 5 |
| 12 | testStrategy present on all P0 requirements | Quality | 5 |
| 13 | Task breakdown hints included | Taskmaster | 5 |

**Total: 65 points**

**Grading:**
- **EXCELLENT:** 91%+ (59+/65)
- **GOOD:** 83-90% (54-58/65)
- **ACCEPTABLE:** 75-82% (49-53/65)
- **NEEDS_WORK:** <75% (<49/65)

**Vague language detection:** Flags words like "fast", "easy", "secure", "scalable", "user-friendly" when used without quantification. Forces specificity.

**If grade < ACCEPTABLE:** PLAN auto-fixes what it can, re-validates, and flags remaining issues to the user.

---

## Phase 5: Sub-Agent Review (4 Fresh Sessions)

After validation passes, PLAN spawns 4 review sub-agents in parallel. Each runs in a **fresh session** (same model) and reviews the PRD from a different angle:

### Reviewer 1: Architecture Review
- Is the proposed architecture sound?
- Are there scalability concerns not addressed?
- Are external dependencies properly handled (failure modes, fallbacks)?
- Does the migration strategy cover rollback?

### Reviewer 2: Requirements Completeness
- Are there gaps in the functional requirements?
- Do acceptance criteria cover edge cases?
- Are error states and failure modes defined?
- Is the dependency chain logically sound?

### Reviewer 3: Scope & Feasibility
- Is the scope realistic for the described system?
- Are there hidden complexities not captured?
- Is the out-of-scope section adequate?
- Do the task breakdown estimates feel reasonable?

### Reviewer 4: testStrategy Review
- Is every P0 requirement's testStrategy actually verifiable?
- Are the verification approaches concrete enough for an autonomous TEST agent?
- Are there requirements that would be hard to test automatically?
- Are performance thresholds measurable?

### Review Aggregation

Each reviewer returns structured feedback:
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

**Decision logic:**
- **All pass:** Proceed to GitHub output
- **Any concerns (no criticals):** PLAN auto-incorporates suggestions, re-validates, proceeds
- **Any critical:** PLAN fixes critical issues, re-runs the specific reviewer that flagged them (max 1 retry), then proceeds or escalates to user

---

## Phase 6: GitHub Output

PLAN creates a PR on the target repo:

```
PR: "plan: [feature-name]"
Branch: plan/[feature-slug]
Labels: plan:draft
```

**Files in PR:**
```
specs/FEAT-[feature-name].md    ← The PRD (comprehensive template)
plans/PLAN-[feature-name].md    ← High-level plan summary (what/why, not how)
```

PLAN also creates a **Plan Issue**:
```
Issue: "[PLAN] Feature Name"
Labels: plan:draft
Body:
  - Executive summary
  - Link to PR
  - Validation score + grade
  - Reviewer verdicts summary
  - Task count estimate
```

---

## Phase 7: Handoff Notification

PLAN messages George via Telegram:

```
PRD ready for [feature-name] — scored GOOD (87%).

• 8 functional requirements (5 P0, 2 P1, 1 P2)
• 6 user stories
• 4 reviewers passed (1 minor suggestion incorporated)
• Estimated ~12 tasks after decomposition

PR: <link>
Plan Issue: <link>

Review and merge when ready. I'll decompose into tasks on next cron run.
```

**PLAN exits.** Interactive mode is a one-shot session.

---

## Phase 8: PR Review Comment Handling (Cron-Driven)

After the PR is created, George may leave review comments before merging. PLAN's cron mode detects and addresses these.

### Detection

On each cron run, PLAN's `find-work.sh` checks:
```bash
# Check for plan:draft PRs with unaddressed review comments
gh pr list --repo "$repo" --label "plan:draft" --state open \
  --json number,title,url,reviews,comments
```

**"Unaddressed" means:** PR has review comments that don't have a PLAN agent reply yet (checked via comment author).

### Response

1. PLAN reads all new review comments on the PR
2. For each comment:
   - If it's a change request: PLAN updates the spec/plan files, pushes a new commit, replies to the comment explaining the change
   - If it's a question: PLAN replies with an answer (or escalates to Telegram if it can't answer)
   - If it's an approval/acknowledgment: PLAN replies with a thumbs-up comment
3. Re-runs validation after changes
4. Notifies George via Telegram: "Updated the [feature] spec PR based on your review comments — [summary of changes]"

### Edge Cases

- **Conflicting comments:** If George's comments contradict each other, PLAN asks for clarification via Telegram rather than guessing
- **Scope-changing comments:** If a comment would significantly change the PRD scope, PLAN flags it: "This would change the scope significantly — want me to revise the whole spec or just note it as a follow-up?"

---

## Post-Merge: Task Decomposition (Cron Mode)

When George merges the spec PR, PLAN's cron mode handles decomposition.

### Taskmaster Integration

1. PLAN detects the merged `plan:draft` PR
2. Copies the spec to a temp working directory with `.taskmaster/docs/prd.md`
3. Runs: `taskmaster parse-prd .taskmaster/docs/prd.md --num-tasks=0 --research`
   - `--num-tasks=0` lets Taskmaster determine optimal task count based on complexity
   - `--research` enables Taskmaster's research mode for better decomposition
4. Runs: `taskmaster expand-all --research` to generate subtasks
5. Runs: `taskmaster analyze-complexity` to get complexity scores
6. For each task in the Taskmaster output:
   - Creates a GitHub issue with:
     - Title, description, and implementation details from Taskmaster
     - testStrategy from the PRD requirement it maps to
     - Complexity score (from Taskmaster's analysis)
     - Dependencies (cross-referenced as issue links)
     - Agent metadata block
   - Labels the issue `ready-for-build`
7. Creates the feature branch: `feat/[feature-slug]`
8. Relabels the plan PR from `plan:draft` to `plan:active`
9. Notifies George: "Decomposed [feature] into N task issues. BUILD will pick them up on next cron."

### Dependency Mapping

Taskmaster outputs dependency IDs (task 3 depends on task 1). PLAN translates these into GitHub issue cross-references:

```markdown
<!-- agent-meta
status: ready-for-build
attempts: 0
max_attempts: 3
feature_branch: feat/add-webhook-handler
spec: specs/FEAT-add-webhook-handler.md
plan: #42
pr:
depends_on: #43,#44
complexity: 7
-->
```

BUILD checks `depends_on` before starting work — if dependent issues aren't `complete`, it skips to the next available task.

---

## Dependencies

- **Runtime:** `task-master-ai` npm package (CLI)
- **Agent:** Servo (for sub-agent spawning in interactive mode)
- **Infrastructure:** OpenClaw cron (for PR comment handling + task decomposition)
- **Target repo:** Must be onboarded (`.agents/commands.yml` exists)

---

## Open Items

1. **Multi-LLM review** — Future enhancement: use different LLM providers for the 4 review sub-agents (tracked as GitHub issue)
2. **Interview adaptive depth** — Should PLAN ask more/fewer questions based on feature complexity? (v2)
3. **PRD versioning** — If George requests major changes after merge, should PLAN create a new spec version or amend? (v2)

---

## Success Criteria

- PLAN can conduct an interview over Telegram and produce a valid PRD in < 15 minutes
- PRD scores ≥ ACCEPTABLE (75%) on automated validation
- All 4 reviewers pass (or concerns are auto-resolved)
- Taskmaster successfully parses the PRD and produces task issues
- George can review and comment on the spec PR, and PLAN addresses feedback without intervention
