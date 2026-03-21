# Blueprint Prompt Improvements

## TL;DR

> **Quick Summary**: Enhance the Blueprint PLAN agent prompt and its DECOMPOSE sub-agent prompt with 10 improvements identified from a comparative analysis against a mature planning system (Prometheus). Improvements target interview adaptivity, research depth, self-regulation, and parallel execution — all as markdown prompt edits, no code changes.
> 
> **Deliverables**:
> - Updated `agents/blueprint/AGENTS.md` with 9 improvements (Intent Classification, Research Prompt Templates, Working Memory, Gap Analysis, Clearance Check, AI Slop Prevention, testStrategy Specificity, Turn Termination, Incremental Output Protocol)
> - Updated `agents/plan/decompose/AGENTS.md` with Parallel Wave support
> - PR submitted with all changes
> 
> **Estimated Effort**: Short
> **Parallel Execution**: YES — 2 waves + verification + PR
> **Critical Path**: Task 1 (Blueprint edits) → Task 3 (Verify) → Task 4 (PR)

---

## Context

### Original Request
Improve the Blueprint PLAN agent prompt with improvements identified from analyzing it against the Prometheus planning system. Submit a PR on completion.

### Analysis Summary
**13 improvements identified, ranked by impact:**
- 5 Critical (intent classification, research templates, working memory, gap analysis, clearance check)
- 5 Medium (AI slop prevention, testStrategy specificity, turn termination, incremental output, parallel waves)
- 3 Lower (structured choices, gap classification, validation merge) — folded into the above 10

**Key Decisions**:
- Phase numbering: ADD SUB-SECTIONS within existing phases (no renumbering, no cascade)
- SETUP.md: NOT in scope (phase numbers unchanged, references stay valid)
- Working Memory: Prompt-level guidance only (not a persisted artifact)
- Gap Analysis escape: Max 1 follow-up iteration, then document in Open Questions
- Branch strategy: Create new branch from current HEAD (`feat/decompose-prompt`)

### Metis Review
**Identified Gaps (addressed):**
- Phase numbering cascade risk → Resolved: use sub-sections, not renumbering
- SETUP.md reference breakage → Resolved: no phase renumbering needed
- Working Memory form factor ambiguity → Resolved: prompt-level guidance only
- Gap Analysis infinite loop → Resolved: max 1 iteration, then proceed with documented gaps
- Prompt length budget → Monitored: target ≤550 lines (from current 381)

---

## Work Objectives

### Core Objective
Make Blueprint a smarter, more adaptive planning agent by adding self-regulation mechanisms (clearance checks, gap analysis), structured research orchestration, and anti-pattern prevention — all within the existing 8-phase pipeline architecture.

### Concrete Deliverables
- `agents/blueprint/AGENTS.md` — enhanced with 9 new sub-sections
- `agents/plan/decompose/AGENTS.md` — enhanced plan comment format with parallel waves

### Definition of Done
- [ ] All 10 improvements verifiable as new content in the respective files
- [ ] Existing phase structure (Phase 1–8) preserved with identical numbering
- [ ] Frozen sections untouched: Phase 5 scoring, Phase 6 reviewer structure, PRD template structure, Output Quality Bar
- [ ] Markdown heading hierarchy valid (no skipped levels)
- [ ] PR created and URL returned

### Must Have
- All 10 improvements present as new sub-sections or enhanced content
- Zero changes to phase numbering (1–8 intact)
- Zero changes to frozen sections (validation scoring, reviewer structure, PRD template, output quality bar)
- PR with descriptive title and body listing all improvements

### Must NOT Have (Guardrails)
- **DO NOT** renumber any existing phases — improvements go IN existing phases as sub-sections
- **DO NOT** modify Phase 5 validation scoring (13 checks, 65 points, 4 grade tiers) — FROZEN
- **DO NOT** modify Phase 6 reviewer structure (4 reviewers, JSON return format, decision logic) — FROZEN
- **DO NOT** modify the PRD template block in Phase 4 (lines 94–197) — FROZEN (but testStrategy Rules section AFTER the template CAN be enhanced)
- **DO NOT** modify the Output Quality Bar (lines 375–381) — FROZEN
- **DO NOT** modify DECOMPOSE Phase 2 (issue creation mechanics, lines 96–229) — FROZEN
- **DO NOT** reword or remove existing Behavioral Rules (lines 357–363) — only ADD new rules
- **DO NOT** touch any files in `agents/plan/reference/`, `agents/blueprint/SETUP.md`, `agents/onboard/`, or `specs/`
- **DO NOT** turn Working Memory into a disk-persisted artifact or new tool — it's prompt guidance only
- **DO NOT** add new validation checks or scoring to Phase 5

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.

### Test Decision
- **Infrastructure exists**: NO (these are markdown prompt files, not code)
- **Automated tests**: None (no test framework applies to prompt files)
- **Framework**: N/A

### QA Policy
Every task includes agent-executed QA scenarios using Bash (grep, diff, wc) to verify content presence and structural integrity. Evidence saved to `.sisyphus/evidence/`.

- **Content verification**: Use Bash (grep) — Search for expected headings and key phrases
- **Structural verification**: Use Bash (diff) — Verify only intended sections changed
- **Integrity verification**: Use Bash (wc, grep) — Line counts, heading hierarchy

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately — both files edited in parallel):
├── Task 1: Blueprint AGENTS.md — all 9 improvements [deep]
└── Task 2: DECOMPOSE AGENTS.md — parallel wave support [quick]

Wave 2 (After Wave 1 — verification):
└── Task 3: Cross-file verification + structural integrity [unspecified-high]

Wave 3 (After Wave 2 — delivery):
└── Task 4: Git branch, commit, PR creation [quick + git-master]

Critical Path: Task 1 → Task 3 → Task 4
Parallel Speedup: ~40% (Wave 1 parallelizes the two file edits)
Max Concurrent: 2 (Wave 1)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 3 | 1 |
| 2 | — | 3 | 1 |
| 3 | 1, 2 | 4 | 2 |
| 4 | 3 | — | 3 |

### Agent Dispatch Summary

- **Wave 1**: **2** — T1 → `deep`, T2 → `quick`
- **Wave 2**: **1** — T3 → `unspecified-high`
- **Wave 3**: **1** — T4 → `quick` + `git-master` skill

---

## TODOs

- [ ] 1. Blueprint AGENTS.md — Add All 9 Prompt Improvements

  **What to do**:

  Edit `agents/blueprint/AGENTS.md` to add 9 new sub-sections within the existing phase structure. Each improvement is a NEW sub-section inserted at a specific location. Do NOT renumber phases. Do NOT modify content outside of the insertion points listed below.

  **CRITICAL**: Read the ENTIRE file first. Understand the structure. Then make targeted insertions. After each insertion, verify the surrounding context is intact.

  **Improvement 1 — Intent Classification (insert after Phase 1, before Phase 2)**:
  Add a new `### Request Classification` sub-section at the END of `## Phase 1: Pre-Interview Codebase Scan` (after line ~37 "This reduces the interview to only what you can't determine from code."). Content:

  ```markdown
  ### Request Classification

  After the codebase scan, classify the request before interviewing:

  - **Trivial** (single endpoint, config change, <10 lines) — Skip full interview. Propose solution, confirm, generate minimal PRD.
  - **Simple** (1-2 components, clear scope, <1 day work) — 1 focused round. Confirm assumptions from scan, ask only critical unknowns.
  - **Standard** (multi-component feature, clear boundaries) — Full 3-round interview as described below.
  - **Complex** (system design, multi-service, architectural impact) — Extended interview (4+ rounds) + deep research phase + architecture review sub-agent.

  This classification determines:
  - Number of interview rounds (0 for trivial, 1 for simple, 3 for standard, 4+ for complex)
  - Research depth (none / light / deep)
  - Whether to use sub-agent reviewers in Phase 6 (skip for trivial/simple)
  - PRD template (minimal for trivial/simple, comprehensive for standard/complex)

  If classification is ambiguous, ask the user: "This could be a quick change or a larger feature. Which feels right?"
  ```

  **Improvement 2 — Working Memory (insert into Phase 2, after Interview Rules)**:
  Add a new `### Interview Working Memory` sub-section at the END of `## Phase 2: Discovery Interview` (after the Interview Rules section, after line ~71). Content:

  ```markdown
  ### Interview Working Memory

  During the interview, maintain a running mental summary of:

  - **Confirmed requirements** — what the user explicitly stated
  - **Technical decisions** — stack choices, integration approaches, patterns
  - **Research needs** — topics requiring Phase 3 investigation
  - **Open questions** — ambiguities not yet resolved
  - **Scope boundaries** — what's IN and what's explicitly OUT

  After each round, briefly restate your understanding to the user: "So far I have: [summary]. Moving to [next topic]." This prevents drift in long conversations and gives the user a chance to correct misunderstandings before they compound.

  For multi-session conversations (user returns hours later), open with: "Picking up where we left off. Here's what we've established: [summary]. Ready to continue?"
  ```

  **Improvement 3 — Clearance Check / Auto-Transition (insert into Phase 2, after Working Memory)**:
  Add a new `### Auto-Transition Rule` sub-section immediately after the Working Memory section. Content:

  ```markdown
  ### Auto-Transition Rule

  After EACH interview round, run this self-check:

  ```
  CLEARANCE CHECK:
  □ Core objective clearly defined?
  □ Scope boundaries established (IN and OUT)?
  □ No critical ambiguities remaining?
  □ Technical approach decided (or inferable from codebase)?
  □ testStrategy approach clear for P0 requirements?
  □ No blocking questions outstanding?
  ```

  - **ALL YES** → Skip remaining rounds. Announce: "I have everything I need. Moving to research and PRD generation." Proceed to Phase 3.
  - **ANY NO** → Continue with the next round, targeting the specific unclear area.
  - **User says "just generate it"** → Respect user agency. Proceed with what you have, documenting gaps in Open Questions. Note: "User chose to proceed with [N] open questions."
  ```

  **Improvement 4 — Research Prompt Templates (REPLACE content within Phase 3)**:
  Replace the existing sparse content of `## Phase 3: Research` (lines ~76-84, the 4 numbered items) with an enhanced version. KEEP the opening sentence "After the interview, always run a research phase before generating the PRD. Do NOT interrupt the user for this phase." Then replace the numbered list with:

  ```markdown
  ### 3.1 — Determine Research Needs

  From the interview, extract research topics:
  - **Codebase questions**: "How is X currently implemented? What patterns exist for Y?"
  - **External questions**: "What API does Z expose? What's the recommended approach for W?"
  - **Gap questions**: "What did the user assume I know that I don't?"

  The depth of research scales with the classification from Phase 1:
  - **Trivial/Simple**: Skip research entirely, or a single quick lookup
  - **Standard**: 1-2 focused sub-agents
  - **Complex**: 3-4 parallel sub-agents covering codebase + external + architecture

  ### 3.2 — Spawn Research Sub-Agents (Parallel)

  **For codebase understanding:**
  > "I'm building [feature] for [repo] and need to match existing codebase conventions exactly. Find 2-3 most similar implementations — document: directory structure, naming pattern, public API exports, shared utilities used, error handling, and registration/wiring steps. Return concrete file paths and patterns, not abstract descriptions."

  **For external APIs/libraries:**
  > "I'm integrating [library/API] and need authoritative implementation guidance. Find official docs: setup, API reference, config options with defaults, pitfalls, and migration gotchas. Also find 1-2 production-quality open-source examples (not tutorials). Return: key API signatures, recommended config, common mistakes."

  **For architecture decisions (complex classification only):**
  > "I'm designing [subsystem] and need to evaluate trade-offs before committing to an approach. Find architectural best practices for this domain: proven patterns, scalability trade-offs, common failure modes, and real-world case studies from engineering blogs. Return: options with pros/cons, recommended approach with rationale."

  ### 3.3 — Synthesize Findings

  Combine all research results into a coherent picture:
  1. Resolve conflicts between sources (prefer official docs over blog posts)
  2. Flag findings that contradict interview assumptions — these become Open Questions
  3. Results feed directly into the PRD's Technical Considerations section
  ```

  **Improvement 5 — Pre-Generation Gap Check (insert at the END of Phase 3, after 3.3)**:
  Add a new sub-section at the very end of Phase 3, before the `---` separator and `## Phase 4`:

  ```markdown
  ### 3.4 — Pre-Generation Gap Check

  Before generating the PRD, review your understanding for completeness:

  1. **Spawn a gap-analysis sub-agent** with your interview summary and research findings:
     > "Review this planning session. The user wants [goal]. We discussed [key points]. Research found [findings]. Identify: questions that should have been asked, guardrails that need setting, assumptions that need validation, missing acceptance criteria, and edge cases not addressed."

  2. **Classify each gap found:**
     - **Critical** (ambiguous business logic, unclear scope boundary) → Ask the user before proceeding
     - **Minor** (missing file path, obvious default) → Self-resolve, note in Open Questions
     - **Ambiguous** (reasonable default exists) → Apply default, document assumption in Open Questions

  3. **Maximum 1 follow-up round with the user** for critical gaps. After that, proceed to Phase 4 with remaining gaps documented in Open Questions. Do not loop indefinitely.
  ```

  **Improvement 6 — AI Slop Prevention (insert into Phase 4, AFTER the testStrategy Rules section)**:
  Add a new `### PRD Anti-Patterns` sub-section AFTER the existing `### testStrategy Rules` section (after line ~211) and BEFORE `## Phase 5`. Content:

  ```markdown
  ### PRD Anti-Patterns (Must Avoid)

  Watch for and prevent these common patterns in generated PRDs:

  - **Scope creep via requirements**: Don't invent requirements the user didn't ask for. If you think something is needed, put it in Open Questions, not Requirements.
  - **Premature architecture**: Don't propose microservices when a function works. Match solution complexity to problem complexity.
  - **Vague testStrategy**: "Verify it works" is NOT a testStrategy. Every testStrategy must name a specific action and expected result. (See testStrategy Rules above.)
  - **Dependency inflation**: Only list BUILD dependencies ("I can't code this without that existing first"). Runtime dependencies (calling another component at execution time) are NOT build dependencies and must not appear in the Dependency Chain.
  - **Gold-plating Non-Functionals**: Don't add "99.99% uptime" and "< 50ms p99" to an internal tool for 3 users. Scale NFR thresholds to the actual context — audience size, criticality, and existing infrastructure.
  - **Over-specified task breakdowns**: Task hints should be directional, not prescriptive. BUILD agents need room to make implementation decisions.
  ```

  **Improvement 7 — Enhance testStrategy Rules (REPLACE existing testStrategy Rules content)**:
  Find the existing `### testStrategy Rules` section (lines ~206-211) and REPLACE its bullet points with an enhanced version. Keep the heading `### testStrategy Rules`. New content:

  ```markdown
  ### testStrategy Rules
  - Every P0 requirement MUST have a `testStrategy` section
  - P1 requirements SHOULD have testStrategy
  - testStrategy must be concrete enough for an autonomous TEST agent to validate without asking questions
  - The TEST agent never sees BUILD's implementation approach — only the spec's testStrategy
  - Avoid vague verification: "check it works" ❌ → "Send POST /api/webhook with payload X, verify 200 response and database row created" ✅

  **Specificity requirements — every testStrategy MUST include:**
  - **Tool**: What validates this (curl, Playwright, bun test, specific CLI command)
  - **Input**: Concrete test data ("test@example.com", not "[email]")
  - **Assertion**: Exact expected result (status 200, body contains field X with value Y — not "returns correct data")
  - **At least ONE failure scenario** per P0 requirement (invalid input, missing auth, rate limit exceeded)

  **Examples:**
  - ❌ BAD: "Verify the login endpoint works correctly"
  - ✅ GOOD: "POST /api/auth/login with {email: 'test@example.com', password: 'validpass'} → 200, response.token is non-empty string. POST with {email: 'test@example.com', password: 'wrong'} → 401, response.error is 'INVALID_CREDENTIALS'."
  ```

  **Improvement 8 — Turn Management (insert into Behavioral Rules section)**:
  Add a new `### Turn Management` sub-section AFTER the existing Behavioral Rules list (after line ~363 "The PRD is the contract...") and BEFORE `## Dependencies`. Content:

  ```markdown
  ### Turn Management

  Every message you send must end with ONE of:
  - **A specific question** to the user (drives conversation forward)
  - **A confirmation + next question** ("Got it — X and Y. Now, about Z...")
  - **An action announcement** ("Running research now. I'll have the PRD ready shortly.")
  - **A handoff summary** (Phase 8 only)

  NEVER end with:
  - "Let me know if you have questions" (passive — puts burden on user)
  - A summary without a follow-up question (dead end)
  - "Ready when you are" (stalls conversation)
  ```

  **Improvement 9 — Large PRD Protocol (insert into Phase 4, after PRD Anti-Patterns)**:
  Add a new `### Large PRD Protocol` sub-section AFTER the PRD Anti-Patterns section and BEFORE `## Phase 5`. Content:

  ```markdown
  ### Large PRD Protocol

  For complex features (>15 requirements or >5 systems involved):

  1. **Generate the PRD incrementally**: Write the skeleton first (all section headers + metadata + Executive Summary), then fill sections one at a time. This prevents output truncation on large documents.
  2. **After completing the PRD**, read it back in full to verify no sections were lost or truncated.
  3. **For standard features**, generate the complete PRD in one pass as usual.
  ```

  **Must NOT do**:
  - Do NOT renumber any existing `## Phase N:` headings
  - Do NOT modify the PRD template block (lines 94–197 between the ``` fences)
  - Do NOT modify Phase 5 validation checks or scoring
  - Do NOT modify Phase 6 reviewer structure or decision logic
  - Do NOT modify the Output Quality Bar section
  - Do NOT modify or remove any existing Behavioral Rules (lines 357–363)
  - Do NOT touch `SETUP.md`, `reference/` files, `onboard/`, or `specs/`

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Complex multi-section editing of a system prompt requires careful reading, precise insertion points, and preservation of surrounding context. Deep analysis needed to avoid breaking existing structure.
  - **Skills**: none required
  - **Skills Evaluated but Omitted**:
    - `frontend-ui-ux`: No UI work
    - `git-master`: No git operations in this task

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 2 — different file)
  - **Parallel Group**: Wave 1 (with Task 2)
  - **Blocks**: Task 3 (verification depends on this completing)
  - **Blocked By**: None (can start immediately)

  **References** (CRITICAL — executor has NO context from this conversation):

  **Pattern References** (existing code to follow):
  - `agents/blueprint/AGENTS.md` — The ENTIRE file. Read it completely before making any changes. Current structure: Phase 1 (lines 28-38), Phase 2 (lines 42-71), Phase 3 (lines 75-84), Phase 4 (lines 88-211), testStrategy Rules (lines 206-211), Phase 5 (lines 215-245), Phase 6 (lines 249-297), Phase 7 (lines 300-329), Phase 8 (lines 333-351), Behavioral Rules (lines 355-363), Dependencies (lines 367-371), Output Quality Bar (lines 375-381)

  **WHY Each Reference Matters**:
  - The full file must be read to understand insertion points — each improvement goes at a specific location relative to existing content
  - The testStrategy Rules section (lines 206-211) is REPLACED with enhanced content
  - The Phase 3 body (lines 76-84, the 4 numbered items) is REPLACED with enhanced content
  - All other improvements are INSERTIONS — new content added without removing existing content

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: All 9 improvement headings exist in file
    Tool: Bash (grep)
    Preconditions: File has been edited with all improvements
    Steps:
      1. grep "### Request Classification" agents/blueprint/AGENTS.md
      2. grep "### Interview Working Memory" agents/blueprint/AGENTS.md
      3. grep "### Auto-Transition Rule" agents/blueprint/AGENTS.md
      4. grep "### 3.1 — Determine Research Needs" agents/blueprint/AGENTS.md
      5. grep "### 3.4 — Pre-Generation Gap Check" agents/blueprint/AGENTS.md
      6. grep "### PRD Anti-Patterns" agents/blueprint/AGENTS.md
      7. grep "### testStrategy Rules" agents/blueprint/AGENTS.md
      8. grep "### Turn Management" agents/blueprint/AGENTS.md
      9. grep "### Large PRD Protocol" agents/blueprint/AGENTS.md
    Expected Result: All 9 grep commands return exactly 1 match each (exit code 0)
    Failure Indicators: Any grep returns 0 matches (missing improvement) or 2+ matches (duplicate)
    Evidence: .sisyphus/evidence/task-1-headings-check.txt

  Scenario: Phase numbering preserved
    Tool: Bash (grep)
    Preconditions: File has been edited
    Steps:
      1. grep -c "## Phase [1-8]:" agents/blueprint/AGENTS.md
    Expected Result: Output is "8" — exactly 8 phase headings, unchanged
    Failure Indicators: Output is not 8 (phases were renumbered, added, or removed)
    Evidence: .sisyphus/evidence/task-1-phase-count.txt

  Scenario: Frozen sections intact
    Tool: Bash (grep)
    Preconditions: File has been edited
    Steps:
      1. grep -c "Total: 65 points" agents/blueprint/AGENTS.md → Expected: 1
      2. grep -c "EXCELLENT: 91%" agents/blueprint/AGENTS.md → Expected: 1
      3. grep -c "GOOD: 83-90%" agents/blueprint/AGENTS.md → Expected: 1
      4. grep -c "ACCEPTABLE: 75-82%" agents/blueprint/AGENTS.md → Expected: 1
      5. grep -c "NEEDS_WORK: <75%" agents/blueprint/AGENTS.md → Expected: 1
      6. grep "A Blueprint PRD should be good enough that:" agents/blueprint/AGENTS.md → Expected: 1 match
    Expected Result: All frozen markers present with correct counts
    Failure Indicators: Any count differs from expected
    Evidence: .sisyphus/evidence/task-1-frozen-sections.txt

  Scenario: testStrategy Rules enhanced (not just preserved)
    Tool: Bash (grep)
    Preconditions: File has been edited
    Steps:
      1. grep "Specificity requirements" agents/blueprint/AGENTS.md
      2. grep "At least ONE failure scenario" agents/blueprint/AGENTS.md
      3. grep "BAD.*Verify the login endpoint" agents/blueprint/AGENTS.md
    Expected Result: All 3 new phrases present (proving enhancement, not just preservation)
    Failure Indicators: Any phrase missing means the enhancement wasn't applied
    Evidence: .sisyphus/evidence/task-1-teststrategy-enhanced.txt

  Scenario: Line count in acceptable range
    Tool: Bash (wc)
    Preconditions: File has been edited
    Steps:
      1. wc -l agents/blueprint/AGENTS.md
    Expected Result: Line count between 450 and 600 (currently 381, adding ~100-200 lines)
    Failure Indicators: Below 400 (content lost) or above 650 (excessive bloat)
    Evidence: .sisyphus/evidence/task-1-line-count.txt
  ```

  **Evidence to Capture:**
  - [ ] task-1-headings-check.txt — grep output for all 9 improvement headings
  - [ ] task-1-phase-count.txt — phase number count verification
  - [ ] task-1-frozen-sections.txt — frozen section marker verification
  - [ ] task-1-teststrategy-enhanced.txt — testStrategy enhancement verification
  - [ ] task-1-line-count.txt — final line count

  **Commit**: YES (group with Task 2)
  - Message: `feat(blueprint): enhance PLAN agent with 10 prompt improvements`
  - Files: `agents/blueprint/AGENTS.md`, `agents/plan/decompose/AGENTS.md`
  - Pre-commit: all QA scenarios pass

- [ ] 2. DECOMPOSE AGENTS.md — Add Parallel Wave Support

  **What to do**:

  Edit `agents/plan/decompose/AGENTS.md` to enhance the plan comment format in Phase 1 (Section 1.4) with parallel execution wave groupings. This is an ENHANCEMENT to the existing plan comment format — the current table format is preserved, and wave groupings are added as additional structure.

  **Specific changes:**

  1. Find the plan comment format section (around lines 67-84) — the markdown table showing `# | Task Title | Size | Depends On | Files Affected`.

  2. ADD a wave grouping section AFTER the existing table in the plan comment format. The new format shows tasks grouped by execution wave:

  ```markdown
  ### Execution Waves

  > Tasks grouped by parallel execution potential. Each wave completes before the next begins.

  **Wave 1** (Start Immediately — no dependencies):
  - Task 1: <title> (small)
  - Task 3: <title> (medium)

  **Wave 2** (After Wave 1):
  - Task 2: <title> → depends on Task 1 (medium)
  - Task 4: <title> → depends on Task 3 (small)

  **Wave 3** (After Wave 2):
  - Task 5: <title> → depends on Tasks 2, 4 (medium)

  Critical Path: Task 1 → Task 2 → Task 5
  Max Parallel: 2 (Waves 1 & 2)
  ```

  3. Also add a note in the `### Notes` section template (line ~83) mentioning that wave groupings are included when task count ≥ 3.

  4. In the `### 1.3 — Decompose into Tasks` section (around line 53), add a 7th bullet to the rules list:
  ```markdown
  7. **Group tasks into parallel execution waves.** Foundation tasks with no dependencies form Wave 1. Tasks depending only on Wave 1 form Wave 2. Continue until all tasks are placed. This helps BUILD agents identify which tasks can run simultaneously.
     - Only generate wave groupings when the decomposition produces 3 or more tasks.
     - A single-task or two-task decomposition does not need wave structure.
  ```

  **Must NOT do**:
  - Do NOT modify Phase 2 (issue creation, backfill, reporting) — lines 96-229 are FROZEN
  - Do NOT change the existing table format — ADD wave groupings alongside it
  - Do NOT modify the Rules section (lines 218-229)
  - Do NOT modify the Input section (lines 9-21)
  - Do NOT change the issue body template (lines 114-154)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single file, single focused improvement, clear insertion points. Quick agent can handle this efficiently.
  - **Skills**: none required

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 1 — different file)
  - **Parallel Group**: Wave 1 (with Task 1)
  - **Blocks**: Task 3 (verification depends on this completing)
  - **Blocked By**: None (can start immediately)

  **References**:

  **Pattern References**:
  - `agents/plan/decompose/AGENTS.md` — The ENTIRE file. Read it completely. Current structure: Input (9-21), Phase 1: Analyze & Plan (24-93), Phase 2: Create Issues (96-229). All changes go in Phase 1 ONLY.
  - `agents/plan/decompose/AGENTS.md:59-84` — Section 1.4 (Post Decomposition Plan Comment) — this is where the wave grouping is added to the plan comment format

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Parallel wave content exists in file
    Tool: Bash (grep)
    Preconditions: File has been edited
    Steps:
      1. grep "Execution Waves\|parallel execution wave\|Wave 1\|Critical Path\|Max Parallel" agents/plan/decompose/AGENTS.md
    Expected Result: Multiple matches confirming wave-related content is present
    Failure Indicators: Zero matches means improvement wasn't applied
    Evidence: .sisyphus/evidence/task-2-wave-content.txt

  Scenario: Phase 2 is untouched
    Tool: Bash (grep)
    Preconditions: File has been edited
    Steps:
      1. grep -c "## Phase 2: Create Issues" agents/plan/decompose/AGENTS.md → Expected: 1
      2. grep -c "### 2.1 — Pass 1: Create Issues" agents/plan/decompose/AGENTS.md → Expected: 1
      3. grep -c "### 2.2 — Pass 2: Backfill Dependencies" agents/plan/decompose/AGENTS.md → Expected: 1
      4. grep -c "### 2.3 — Report Results" agents/plan/decompose/AGENTS.md → Expected: 1
    Expected Result: All Phase 2 headings present with count 1 each
    Failure Indicators: Any count differs (Phase 2 was modified)
    Evidence: .sisyphus/evidence/task-2-phase2-intact.txt

  Scenario: Line count in acceptable range
    Tool: Bash (wc)
    Preconditions: File has been edited
    Steps:
      1. wc -l agents/plan/decompose/AGENTS.md
    Expected Result: Between 260 and 320 lines (currently 229, adding ~40-80 lines)
    Failure Indicators: Below 225 (content lost) or above 350 (excessive bloat)
    Evidence: .sisyphus/evidence/task-2-line-count.txt
  ```

  **Evidence to Capture:**
  - [ ] task-2-wave-content.txt — wave-related content grep
  - [ ] task-2-phase2-intact.txt — Phase 2 preservation verification
  - [ ] task-2-line-count.txt — final line count

  **Commit**: YES (grouped with Task 1)
  - Message: `feat(blueprint): enhance PLAN agent with 10 prompt improvements`
  - Files: `agents/blueprint/AGENTS.md`, `agents/plan/decompose/AGENTS.md`

- [ ] 3. Cross-File Verification and Structural Integrity Check

  **What to do**:

  After Tasks 1 and 2 complete, verify that both files are structurally sound and all 10 improvements are present. This is a READ-ONLY verification task — do NOT edit any files.

  1. **Read `agents/blueprint/AGENTS.md` completely** and verify:
     - All 8 original `## Phase N:` headings are present and correctly numbered (1-8)
     - All 9 new sub-section headings exist at the correct locations within their parent phases
     - Markdown heading hierarchy is valid: `#` → `##` → `###` (no skipped levels)
     - No orphaned code blocks (every ``` has a matching ```)
     - Frozen sections are byte-identical to originals: Phase 5 scoring, Phase 6 reviewers, PRD template, Output Quality Bar, existing Behavioral Rules

  2. **Read `agents/plan/decompose/AGENTS.md` completely** and verify:
     - Phase 1 contains the new wave grouping content
     - Phase 2 is completely unchanged
     - Rules section is completely unchanged
     - Markdown structure is valid

  3. **Run all verification commands** from Success Criteria section and save output as evidence.

  4. **If ANY verification fails**: Report which specific check failed and what the actual vs expected value is. Do NOT attempt to fix — just report.

  **Must NOT do**:
  - Do NOT edit any files — this is a read-only verification task
  - Do NOT create any new files (except evidence files in `.sisyphus/evidence/`)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Needs thorough attention to detail for verification across two files. Not complex logic, but requires care.
  - **Skills**: none required

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (solo)
  - **Blocks**: Task 4 (PR creation depends on verification passing)
  - **Blocked By**: Task 1 AND Task 2

  **References**:
  - `agents/blueprint/AGENTS.md` — Read the full edited file
  - `agents/plan/decompose/AGENTS.md` — Read the full edited file

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: All 10 improvements verified present
    Tool: Bash (grep)
    Preconditions: Tasks 1 and 2 are complete
    Steps:
      1. Run ALL grep commands from Success Criteria section:
         grep -c "Request Classification\|Interview Working Memory\|Auto-Transition\|3.1 — Determine Research\|3.4 — Pre-Generation Gap\|PRD Anti-Patterns\|testStrategy Rules\|Turn Management\|Large PRD Protocol" agents/blueprint/AGENTS.md
      2. grep -c "Execution Waves\|parallel execution wave" agents/plan/decompose/AGENTS.md
      3. grep -c "## Phase [1-8]:" agents/blueprint/AGENTS.md → Expected: 8
      4. grep -c "Total: 65 points" agents/blueprint/AGENTS.md → Expected: 1
      5. wc -l agents/blueprint/AGENTS.md → Expected: 450-600
      6. wc -l agents/plan/decompose/AGENTS.md → Expected: 260-320
    Expected Result: All checks pass with expected values
    Failure Indicators: Any check returns unexpected value
    Evidence: .sisyphus/evidence/task-3-full-verification.txt

  Scenario: Markdown heading hierarchy valid
    Tool: Bash (grep)
    Preconditions: Tasks 1 and 2 are complete
    Steps:
      1. grep "^#" agents/blueprint/AGENTS.md | head -50 — verify headings appear in logical order
      2. grep "^#" agents/plan/decompose/AGENTS.md | head -20 — verify headings appear in logical order
    Expected Result: Headings follow # → ## → ### hierarchy without skips
    Failure Indicators: A ### appears without a preceding ##, or heading order is illogical
    Evidence: .sisyphus/evidence/task-3-heading-hierarchy.txt
  ```

  **Evidence to Capture:**
  - [ ] task-3-full-verification.txt — complete verification output
  - [ ] task-3-heading-hierarchy.txt — heading hierarchy audit

  **Commit**: NO

- [ ] 4. Create Git Branch, Commit, and Submit PR

  **What to do**:

  Create a feature branch, commit the changes, and submit a PR.

  1. **Check current state**: Run `git status` and `git branch` to understand current position.

  2. **Create feature branch**: `git checkout -b feat/blueprint-prompt-improvements` (from current HEAD)

  3. **Stage files**: `git add agents/blueprint/AGENTS.md agents/plan/decompose/AGENTS.md`

  4. **Commit**: With message `feat(blueprint): enhance PLAN agent with 10 prompt improvements`

  5. **Push**: `git push origin feat/blueprint-prompt-improvements`

  6. **Create PR** using `gh pr create`:
     - Title: `feat(blueprint): enhance PLAN agent with 10 prompt improvements`
     - Base: `main`
     - Body (use HEREDOC):

     ```
     ## Summary
     - Enhance Blueprint PLAN agent prompt with 9 improvements to interview adaptivity, research depth, and self-regulation
     - Enhance DECOMPOSE sub-agent with parallel execution wave support in task decomposition

     ## Improvements Added to `agents/blueprint/AGENTS.md`

     | # | Improvement | Location | Type |
     |---|------------|----------|------|
     | 1 | Request Classification | Phase 1 | New sub-section |
     | 2 | Interview Working Memory | Phase 2 | New sub-section |
     | 3 | Auto-Transition Rule | Phase 2 | New sub-section |
     | 4 | Research Prompt Templates | Phase 3 | Enhanced content |
     | 5 | Pre-Generation Gap Check | Phase 3 | New sub-section |
     | 6 | PRD Anti-Patterns | Phase 4 | New sub-section |
     | 7 | testStrategy Rules | Phase 4 | Enhanced content |
     | 8 | Turn Management | Behavioral Rules | New sub-section |
     | 9 | Large PRD Protocol | Phase 4 | New sub-section |

     ## Improvements Added to `agents/plan/decompose/AGENTS.md`

     | # | Improvement | Location | Type |
     |---|------------|----------|------|
     | 10 | Parallel Execution Waves | Phase 1 (Sections 1.3, 1.4) | Enhanced content |

     ## What Did NOT Change (Frozen)
     - Phase numbering (1-8) — unchanged
     - Phase 5 validation scoring (13 checks, 65 points) — untouched
     - Phase 6 reviewer structure (4 reviewers) — untouched
     - PRD template block — untouched
     - Output Quality Bar — untouched
     - DECOMPOSE Phase 2 (issue creation) — untouched
     - SETUP.md — untouched
     - Reference files — untouched
     ```

  7. **Return the PR URL** as the final output.

  **Must NOT do**:
  - Do NOT push to `main` directly
  - Do NOT use `--force` push
  - Do NOT commit any files other than the two listed
  - Do NOT amend any existing commits

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Standard git operations — branch, commit, push, PR
  - **Skills**: [`git-master`]
    - `git-master`: Git operations and PR creation

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3 (solo)
  - **Blocks**: None (final task)
  - **Blocked By**: Task 3

  **References**:
  - `agents/blueprint/AGENTS.md` — file to commit
  - `agents/plan/decompose/AGENTS.md` — file to commit

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: PR created successfully
    Tool: Bash (gh)
    Preconditions: Changes committed and pushed
    Steps:
      1. gh pr view --json url,title,state
    Expected Result: PR exists with correct title, state is "OPEN"
    Failure Indicators: gh pr view fails or state is not OPEN
    Evidence: .sisyphus/evidence/task-4-pr-created.txt

  Scenario: PR targets correct base and includes both files
    Tool: Bash (gh)
    Preconditions: PR exists
    Steps:
      1. gh pr view --json baseRefName -q .baseRefName → Expected: "main"
      2. gh pr diff --name-only → Expected: both AGENTS.md files listed
    Expected Result: PR targets main, diff shows exactly 2 files
    Failure Indicators: Wrong base branch or unexpected files in diff
    Evidence: .sisyphus/evidence/task-4-pr-details.txt
  ```

  **Evidence to Capture:**
  - [ ] task-4-pr-created.txt — PR URL and metadata
  - [ ] task-4-pr-details.txt — base branch and file list

  **Commit**: YES (this IS the commit task)
  - Message: `feat(blueprint): enhance PLAN agent with 10 prompt improvements`
  - Files: `agents/blueprint/AGENTS.md`, `agents/plan/decompose/AGENTS.md`
  - Pre-commit: Task 3 verification passed

---

## Final Verification Wave

> This plan is small enough that the Final Verification Wave is replaced by Task 3 (inline verification) and Task 4's PR review body which lists all 10 improvements for human reviewability.

---

## Commit Strategy

- **1**: `feat(blueprint): enhance PLAN agent with 10 prompt improvements` — `agents/blueprint/AGENTS.md`, `agents/plan/decompose/AGENTS.md`

---

## Success Criteria

### Verification Commands
```bash
# All 10 improvement headings exist
grep -c "Request Classification\|Interview Working Memory\|Auto-Transition\|Determine Research Needs\|Pre-Generation Gap Check\|PRD Anti-Patterns\|testStrategy Rules\|Turn Management\|Large PRD Protocol" agents/blueprint/AGENTS.md  # Expected: 9
grep -c "Execution Waves" agents/plan/decompose/AGENTS.md  # Expected: 1

# Phase numbers unchanged
grep -c "## Phase [1-8]:" agents/blueprint/AGENTS.md  # Expected: 8

# Frozen sections intact
grep -c "Total: 65 points" agents/blueprint/AGENTS.md  # Expected: 1
grep -c "EXCELLENT: 91%+" agents/blueprint/AGENTS.md  # Expected: 1

# Line count in range
wc -l agents/blueprint/AGENTS.md  # Expected: 450-600
wc -l agents/plan/decompose/AGENTS.md  # Expected: 260-320
```

### Final Checklist
- [ ] All 10 improvements present as verifiable headings/content
- [ ] Phase 1–8 numbering preserved
- [ ] Frozen sections unchanged
- [ ] Markdown structure valid
- [ ] PR created with descriptive body
- [ ] All tests pass (verification commands above)
