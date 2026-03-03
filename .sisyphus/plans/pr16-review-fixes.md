# PR #16 Review Fixes — Elevate Agent Prompts to Production Quality

## TL;DR

> **Quick Summary**: Fix 15 issues (3 critical, 4 high, 4 medium, 4 low) identified in PR #16's agent prompt improvements. All changes are surgical markdown edits to 3 files — no code, no new features.
> 
> **Deliverables**:
> - `specs/FEAT-agent-workflow-system.md` — Updated metadata schema (three-counter alignment)
> - `agents/plan/decompose/AGENTS.md` — Robust backfill, recovery, failure handling, wave validation
> - `agents/blueprint/AGENTS.md` — Reframed working memory, softened boundaries, non-API testStrategy, gap check limits
> 
> **Estimated Effort**: Short (1-2 hours agent time)
> **Parallel Execution**: YES — 3 waves
> **Critical Path**: Task 1 (schema alignment) → Task 5 (backfill recovery) → Task 10 (cross-file verification)

---

## Context

### Original Request
Fix all 15 issues identified in code review of PR #16 (`feat/blueprint-prompt-improvements`) on AIpexForge/agent-swarm. The PR enhances Blueprint (PLAN agent) prompts with 9 improvements and adds a new DECOMPOSE sub-agent for task decomposition.

### Interview Summary
**Key Discussions**:
- PR #16 is on branch `feat/blueprint-prompt-improvements`, already has `main` merged in
- All changes are markdown — agent prompts and specifications
- Issues range from schema mismatches (critical) to nice-to-have edge cases (low)
- oh-my-opencode patterns are the gold standard for comparison

**Research Findings**:
- Three-counter metadata schema (`attempts_build/test/review`) originated in archived spec `FEAT-plan-task-decomposition.md`
- Backfill recovery rules (check for `depends_on: pending` on retry) exist in archived spec line 385 but were not carried into DECOMPOSE prompt
- `specs/FEAT-plan-interactive-mode.md` also uses the old single-counter schema (additional divergence beyond what review caught)
- Workflow spec has `status:` field in metadata that DECOMPOSE omits (intentional — status tracked by labels)

### Metis Review
**Identified Gaps** (addressed):
- Interactive mode spec (`FEAT-plan-interactive-mode.md`) also has stale metadata schema — added to scope as part of C1 fix
- `status:` field omission in DECOMPOSE is intentional (labels track state) — documented, no fix needed
- Schema direction decision: three-counter is canonical (most evolved, per-stage tracking) — workflow spec adopts it
- Edit collision risk for DECOMPOSE — tasks sequenced to avoid overlapping sections
- `find-work.py` vs `.sh` naming: PLAN uses `.py` (correct per task decomp spec), other agents use `.sh` — fix is in workflow spec only

---

## Work Objectives

### Core Objective
Resolve all 15 review issues so PR #16's agent prompts are production-quality and internally consistent across the spec ecosystem.

### Concrete Deliverables
- `specs/FEAT-agent-workflow-system.md` — §Issue Metadata updated to three-counter schema
- `agents/plan/decompose/AGENTS.md` — 7 issues fixed (C2, C3, H1, H4, L1, L2, L4)
- `agents/blueprint/AGENTS.md` — 6 issues fixed (H2, H3, M1, M2, M3, M4)

### Definition of Done
- [ ] `grep -c "attempts: 0" specs/FEAT-agent-workflow-system.md` returns 0
- [ ] `grep -c "attempts_build: 0" specs/FEAT-agent-workflow-system.md` returns 1
- [ ] No literal `sed` command in `agents/plan/decompose/AGENTS.md`
- [ ] DECOMPOSE Rules section has ≥12 rules (was 10, adding recovery + retry + wave validation)
- [ ] Blueprint AGENTS.md line count ≥ 530 (was 509, adding ~30 lines of fixes)
- [ ] All markdown renders without broken formatting

### Must Have
- Three-counter metadata schema consistent across workflow spec and DECOMPOSE
- Backfill mechanism uses structured approach (not raw sed)
- Backfill recovery rule in DECOMPOSE (from archived spec)
- Failure recovery protocol in DECOMPOSE
- Wave ordering validation rule

### Must NOT Have (Guardrails)
- **No new agent capabilities** — these are prompt fixes, not feature additions
- **No touching archived specs** (`specs/archive/*`) — reference only
- **No SETUP.md changes** — frozen per PR description
- **No reference file changes** (`agents/plan/reference/*`) — frozen per PR description
- **No full rewrites** — surgical patches to specific sections only
- **No scope creep into BUILD/TEST/REVIEW agent prompts** — those don't exist yet
- **Do NOT add oh-my-opencode-specific patterns** (like TodoWrite) that don't fit this system's architecture

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.

### Test Decision
- **Infrastructure exists**: NO (markdown-only repo)
- **Automated tests**: None
- **Framework**: N/A

### QA Policy
Every task MUST include agent-executed QA scenarios.
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Schema consistency**: Use Bash (grep) — Search for old/new field names across all .md files
- **Markdown integrity**: Use Bash — Verify line counts, heading structure, no broken formatting
- **Content verification**: Use Read tool — Confirm specific text appears in expected locations

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately — independent file edits, no cross-dependencies):
├── Task 1: [C1] Align metadata schema in workflow spec         [quick]
├── Task 2: [H2] Reframe interview working memory              [quick]
├── Task 3: [H3] Add non-API testStrategy format               [quick]
├── Task 4: [M1+M2] Soften classification + cap gap check      [quick]
├── Task 5: [M3+M4] Delegation structure + PRD tracking         [quick]

Wave 2 (After Wave 1 — DECOMPOSE edits, sequenced to avoid collision):
├── Task 6: [C2] Replace sed backfill with robust approach      [quick]
├── Task 7: [C3+H1] Add recovery + failure protocol to Rules    [quick]
├── Task 8: [H4+L1+L4] Wave validation + edge cases to Rules   [quick]
├── Task 9: [L2+L3] Spec versioning + naming consistency       [quick]

Wave 3 (After Wave 2 — cross-file verification):
├── Task 10: Cross-file consistency verification                [quick]

Wave FINAL (After ALL tasks — independent review):
├── Task F1: Plan compliance audit                              [unspecified-high]
├── Task F2: Markdown quality + rendering check                 [unspecified-high]
├── Task F3: Schema consistency grep                            [quick]
├── Task F4: Scope fidelity check                               [deep]

Critical Path: Task 1 → Task 6 → Task 10 → F1-F4
Parallel Speedup: ~60% faster than sequential
Max Concurrent: 5 (Wave 1)
```

### Dependency Matrix

| Task | Depends On | Blocks |
|------|-----------|--------|
| 1 | — | 6, 9, 10 |
| 2 | — | 10 |
| 3 | — | 10 |
| 4 | — | 10 |
| 5 | — | 10 |
| 6 | 1 | 10 |
| 7 | — | 10 |
| 8 | — | 10 |
| 9 | 1 | 10 |
| 10 | 2, 3, 4, 5, 6, 7, 8, 9 | F1-F4 |
| F1-F4 | 10 | — |

### Agent Dispatch Summary

- **Wave 1**: 5 tasks → all `quick`
- **Wave 2**: 4 tasks → all `quick`
- **Wave 3**: 1 task → `quick`
- **FINAL**: 4 tasks → F1 `unspecified-high`, F2 `unspecified-high`, F3 `quick`, F4 `deep`

---

## TODOs

- [ ] 1. [C1] Align metadata schema in workflow spec

  **What to do**:
  - Open `specs/FEAT-agent-workflow-system.md`
  - Replace the §Issue Metadata block (lines ~266-284) to use three-counter schema:
    ```
    <!-- agent-meta
    attempts_build: 0
    attempts_test: 0
    attempts_review: 0
    max_attempts: 3
    size: small
    feature_branch: feat/add-webhook-handler
    spec: specs/FEAT-add-webhook-handler.md
    plan: #42
    pr:
    depends_on:
    -->
    ```
  - Update the field descriptions below the block:
    - Remove: `- \`attempts\` — Number of build/test/review cycles`
    - Add: `- \`attempts_build\` — BUILD agent attempt count`, `- \`attempts_test\` — TEST agent attempt count`, `- \`attempts_review\` — REVIEW agent attempt count`
  - Add `depends_on` field description: `- \`depends_on\` — Comma-separated issue refs (e.g., #43,#44) or empty`
  - Add `size` field description: `- \`size\` — Task size estimate (small or medium)`

  **Must NOT do**:
  - Do NOT touch any other section of the workflow spec
  - Do NOT modify the `status` field descriptions
  - Do NOT add fields not in the DECOMPOSE template

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single file, find-and-replace in one section
  - **Skills**: []
    - No specialized skills needed for markdown editing

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3, 4, 5)
  - **Blocks**: Tasks 6, 9, 10
  - **Blocked By**: None (can start immediately)

  **References**:

  **Pattern References**:
  - `specs/archive/FEAT-plan-task-decomposition.md:605-614` — Canonical three-counter schema with field descriptions
  - `agents/plan/decompose/AGENTS.md:138-148` — DECOMPOSE's metadata template (target to match)

  **API/Type References**:
  - `specs/FEAT-agent-workflow-system.md:266-284` — Current (outdated) metadata block to replace

  **WHY Each Reference Matters**:
  - The archived spec is the source of truth for the evolved schema — copy field names exactly
  - DECOMPOSE template is what issues will actually contain — workflow spec must match it

  **Acceptance Criteria**:
  - [ ] `grep -c "attempts: 0" specs/FEAT-agent-workflow-system.md` → 0
  - [ ] `grep -c "attempts_build: 0" specs/FEAT-agent-workflow-system.md` → 1
  - [ ] `grep -c "depends_on:" specs/FEAT-agent-workflow-system.md` → ≥1
  - [ ] `grep -c "size:" specs/FEAT-agent-workflow-system.md` → ≥1

  **QA Scenarios**:

  ```
  Scenario: Schema fields match DECOMPOSE template
    Tool: Bash (grep)
    Preconditions: Task 1 edits applied
    Steps:
      1. grep -n "attempts_build: 0" specs/FEAT-agent-workflow-system.md
      2. grep -n "attempts_test: 0" specs/FEAT-agent-workflow-system.md
      3. grep -n "attempts_review: 0" specs/FEAT-agent-workflow-system.md
      4. grep -n "depends_on:" specs/FEAT-agent-workflow-system.md
      5. grep -n "size:" specs/FEAT-agent-workflow-system.md
    Expected Result: All 5 greps return exactly 1 match each within the <!-- agent-meta --> block
    Failure Indicators: Any grep returns 0 matches or matches outside the metadata block
    Evidence: .sisyphus/evidence/task-1-schema-alignment.txt

  Scenario: Old single-counter schema fully removed
    Tool: Bash (grep)
    Preconditions: Task 1 edits applied
    Steps:
      1. grep -n "^attempts: " specs/FEAT-agent-workflow-system.md (expect no match)
      2. grep -c "Number of build/test/review cycles" specs/FEAT-agent-workflow-system.md (expect 0)
    Expected Result: Zero matches for old schema patterns
    Failure Indicators: Any match found for old `attempts:` field or description
    Evidence: .sisyphus/evidence/task-1-old-schema-removed.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Message: `fix(agents): resolve 15 review issues in PR #16 prompt improvements`
  - Files: `specs/FEAT-agent-workflow-system.md`

- [ ] 2. [H2] Reframe interview working memory in Blueprint

  **What to do**:
  - Open `agents/blueprint/AGENTS.md`
  - Replace the "Interview Working Memory" section (lines ~90-102) with actionable output format:
    - Change opening from "maintain a running mental summary of" to "Before each response, review the conversation and produce a brief status block:"
    - Add concrete format:
      ```
      STATUS: Round [N] of [max]
      CONFIRMED: [bullet list of confirmed requirements]
      DECIDED: [technical decisions made]
      OPEN: [unresolved questions]
      SCOPE: IN=[list] | OUT=[list]
      NEXT: [what you'll ask about next]
      ```
    - Keep the "After each round, briefly restate" instruction but anchor it to this format
    - Keep the multi-session resumption instruction ("Picking up where we left off...")

  **Must NOT do**:
  - Do NOT change Phase 2 interview rounds structure
  - Do NOT modify Auto-Transition Rule section
  - Do NOT change the 5 categories (just reframe how they're surfaced)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single section rewrite in one file
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3, 4, 5)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `agents/blueprint/AGENTS.md:90-102` — Current working memory section to reframe

  **External References**:
  - oh-my-opencode's draft management pattern — externalizes state via structured files rather than relying on implicit LLM memory

  **WHY Each Reference Matters**:
  - The current section tells the LLM to "maintain a mental summary" which isn't actionable — the replacement gives it a concrete output format to produce

  **Acceptance Criteria**:
  - [ ] "STATUS:" format block appears in the Interview Working Memory section
  - [ ] "maintain a running mental summary" phrase is removed
  - [ ] Multi-session resumption instruction preserved
  - [ ] "After each round, briefly restate" instruction preserved

  **QA Scenarios**:

  ```
  Scenario: Working memory section has concrete format
    Tool: Bash (grep)
    Preconditions: Task 2 edits applied
    Steps:
      1. grep -c "STATUS: Round" agents/blueprint/AGENTS.md
      2. grep -c "CONFIRMED:" agents/blueprint/AGENTS.md
      3. grep -c "maintain a running mental summary" agents/blueprint/AGENTS.md
      4. grep -c "Picking up where we left off" agents/blueprint/AGENTS.md
    Expected Result: Steps 1-2 return ≥1, step 3 returns 0, step 4 returns 1
    Failure Indicators: Old aspirational phrasing still present, or resumption instruction missing
    Evidence: .sisyphus/evidence/task-2-working-memory.txt

  Scenario: Section doesn't break surrounding content
    Tool: Bash (grep)
    Preconditions: Task 2 edits applied
    Steps:
      1. grep -n "### Interview Working Memory" agents/blueprint/AGENTS.md
      2. grep -n "### Auto-Transition Rule" agents/blueprint/AGENTS.md
      3. Verify Auto-Transition section follows Working Memory with no missing headings
    Expected Result: Both headings exist, Auto-Transition follows Working Memory
    Failure Indicators: Either heading missing or out of order
    Evidence: .sisyphus/evidence/task-2-heading-integrity.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/blueprint/AGENTS.md`

- [ ] 3. [H3] Add non-API testStrategy format to Blueprint

  **What to do**:
  - Open `agents/blueprint/AGENTS.md`
  - After the testStrategy examples (line ~307, after the ✅ GOOD example), add a new subsection:
    ```
    **For non-API requirements** (UI, migrations, infrastructure, config), adapt the format:
    - **Tool**: What validates this (Playwright for UI, migration script dry-run, healthcheck endpoint)
    - **Setup**: Required preconditions/state (seed data loaded, feature flag enabled, clean database)
    - **Action**: What to do (navigate to /settings, run migration, restart service)
    - **Assertion**: Observable result (element `.dark-mode-toggle` visible, table `users` has column `mfa_enabled`, service responds 200 on /health within 30s)
    - **Rollback**: How to undo if it fails (revert migration, restore config)
    
    **Examples:**
    - ❌ BAD: "Verify the dark mode toggle works"
    - ✅ GOOD: "Playwright: Navigate to /settings, click `[data-testid=theme-toggle]`, assert `document.body` has class `dark`. Screenshot evidence. Revert: click toggle again, assert class `light`."
    ```

  **Must NOT do**:
  - Do NOT modify existing API testStrategy examples
  - Do NOT change the specificity requirements list (Tool/Input/Assertion/Failure)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Additive section, no existing content modified
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 4, 5)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `agents/blueprint/AGENTS.md:299-307` — Existing testStrategy rules and examples to extend

  **WHY Each Reference Matters**:
  - Must match the tone and format of existing examples — additive, not disruptive

  **Acceptance Criteria**:
  - [ ] "non-API requirements" subsection exists after existing testStrategy examples
  - [ ] Contains Playwright example for UI
  - [ ] Contains migration example
  - [ ] Original API examples unchanged

  **QA Scenarios**:

  ```
  Scenario: Non-API testStrategy section present
    Tool: Bash (grep)
    Preconditions: Task 3 edits applied
    Steps:
      1. grep -c "non-API requirements" agents/blueprint/AGENTS.md
      2. grep -c "Playwright:" agents/blueprint/AGENTS.md (in testStrategy context)
      3. grep -c "Rollback:" agents/blueprint/AGENTS.md
    Expected Result: All return ≥1
    Failure Indicators: Any return 0
    Evidence: .sisyphus/evidence/task-3-non-api-teststrategy.txt

  Scenario: Existing API examples preserved
    Tool: Bash (grep)
    Preconditions: Task 3 edits applied
    Steps:
      1. grep -c "POST /api/auth/login" agents/blueprint/AGENTS.md
      2. grep -c "INVALID_CREDENTIALS" agents/blueprint/AGENTS.md
    Expected Result: Both return 1 (original examples intact)
    Failure Indicators: Either return 0 (originals were overwritten)
    Evidence: .sisyphus/evidence/task-3-api-preserved.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/blueprint/AGENTS.md`

- [ ] 4. [M1+M2] Soften classification boundaries + cap gap check

  **What to do**:
  - Open `agents/blueprint/AGENTS.md`
  - **M1 — Classification boundaries** (lines ~44-55):
    - Remove hard numeric thresholds from classification tiers. Change `<10 lines` to `~10 lines or fewer`. Change `<1 day work` to `~1 day or less`.
    - After the existing escape hatch (line 55: "If classification is ambiguous..."), add: "These are sizing guidelines, not hard rules. A 15-line config change is still trivial. A 50-line copy-paste migration is still simple. Use judgment — classify by *cognitive complexity*, not line count."
  - **M2 — Gap check loop protection** (§3.4, lines ~158-170):
    - After "Maximum 1 follow-up round with the user" (line 170), add: "If the gap agent returns more than 3 critical gaps, this signals an insufficient interview — not a gap-check problem. Return to Phase 2 for one focused round targeting the top 3 gaps. Document remaining gaps in Open Questions. Do not attempt to resolve 4+ critical gaps via follow-up text."

  **Must NOT do**:
  - Do NOT remove the classification tiers themselves (Trivial/Simple/Standard/Complex)
  - Do NOT change the "determines" list (rounds, depth, reviewers, template)
  - Do NOT modify Phase 3 research templates

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Two small edits in the same file, no structural changes
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3, 5)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `agents/blueprint/AGENTS.md:44-55` — Classification section to soften
  - `agents/blueprint/AGENTS.md:158-170` — Gap check section to cap

  **WHY Each Reference Matters**:
  - Classification thresholds create edge-case arguments; softening them while keeping the tiers preserves structure with flexibility
  - Gap check without a cap could dump 20 critical items on a user, defeating the max-1-follow-up intent

  **Acceptance Criteria**:
  - [ ] No literal `<10 lines` in classification section (replaced with `~10 lines or fewer`)
  - [ ] "cognitive complexity" phrase present in classification section
  - [ ] "more than 3 critical gaps" rule present in §3.4
  - [ ] "Return to Phase 2" instruction present for excessive gaps

  **QA Scenarios**:

  ```
  Scenario: Classification boundaries softened
    Tool: Bash (grep)
    Preconditions: Task 4 edits applied
    Steps:
      1. grep -c "<10 lines" agents/blueprint/AGENTS.md (expect 0)
      2. grep -c "cognitive complexity" agents/blueprint/AGENTS.md (expect 1)
      3. grep -c "~10 lines" agents/blueprint/AGENTS.md (expect 1)
    Expected Result: Step 1 → 0, Steps 2-3 → 1 each
    Failure Indicators: Hard thresholds still present or softening text missing
    Evidence: .sisyphus/evidence/task-4-classification.txt

  Scenario: Gap check has loop protection
    Tool: Bash (grep)
    Preconditions: Task 4 edits applied
    Steps:
      1. grep -c "more than 3 critical gaps" agents/blueprint/AGENTS.md (expect 1)
      2. grep -c "Return to Phase 2" agents/blueprint/AGENTS.md (expect 1)
    Expected Result: Both return 1
    Failure Indicators: Gap check section unchanged
    Evidence: .sisyphus/evidence/task-4-gap-check.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/blueprint/AGENTS.md`

- [ ] 5. [M3+M4] Standardize delegation structure + add PRD tracking

  **What to do**:
  - Open `agents/blueprint/AGENTS.md`
  - **M3 — Sub-agent delegation structure**: After §3.2's research templates (around line ~149), add a note:
    ```
    ### Sub-Agent Prompt Structure
    
    All sub-agent prompts (research, validation, review, gap-check, GitHub output) should follow this structure:
    - **Task**: What specifically to do (1-2 sentences)
    - **Context**: Background the sub-agent needs (interview findings, repo info)
    - **Expected Output**: What to return (format, fields, structure)
    - **Must NOT**: Explicit constraints (don't execute code, don't hallucinate, don't modify files)
    
    The research templates in §3.2 demonstrate this pattern. Apply it consistently to Phase 5 (validation), Phase 6 (review), and Phase 7 (GitHub output) sub-agent invocations.
    ```
  - **M4 — Large PRD tracking**: In the "Large PRD Protocol" section (lines ~320-327), after "fill sections one at a time", add: "Track completion by working through sections in this order: Executive Summary → Problem Statement → Goals → User Stories → P0 Requirements → P1 Requirements → P2 Requirements → NFRs → Technical Considerations → Implementation Roadmap → Dependency Chain → Out of Scope → Open Questions → Appendix. After each section, read the document from the top to verify no prior sections were lost or corrupted."

  **Must NOT do**:
  - Do NOT rewrite existing research templates to match the structure (they already mostly do)
  - Do NOT add a formal schema/interface for sub-agent prompts
  - Do NOT change Phase 5/6/7 content (just reference the pattern)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Two additive insertions in one file
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3, 4)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `agents/blueprint/AGENTS.md:140-149` — Research templates (already demonstrate the pattern)
  - `agents/blueprint/AGENTS.md:320-327` — Large PRD Protocol to extend

  **WHY Each Reference Matters**:
  - Research templates are the best existing example of structured delegation — the new note codifies what they already do
  - Large PRD Protocol lacks section-ordering guidance — adding it prevents silent section loss

  **Acceptance Criteria**:
  - [ ] "Sub-Agent Prompt Structure" heading exists in Phase 3 area
  - [ ] Contains "Task", "Context", "Expected Output", "Must NOT" fields
  - [ ] Large PRD Protocol includes ordered section list
  - [ ] "read the document from the top" instruction present

  **QA Scenarios**:

  ```
  Scenario: Delegation structure documented
    Tool: Bash (grep)
    Preconditions: Task 5 edits applied
    Steps:
      1. grep -c "Sub-Agent Prompt Structure" agents/blueprint/AGENTS.md (expect 1)
      2. grep -c "Expected Output" agents/blueprint/AGENTS.md (expect ≥1)
    Expected Result: Both return ≥1
    Failure Indicators: Section missing
    Evidence: .sisyphus/evidence/task-5-delegation.txt

  Scenario: Large PRD has section ordering
    Tool: Bash (grep)
    Preconditions: Task 5 edits applied
    Steps:
      1. grep -c "Executive Summary → Problem Statement" agents/blueprint/AGENTS.md (expect 1)
      2. grep -c "read the document from the top" agents/blueprint/AGENTS.md (expect 1)
    Expected Result: Both return 1
    Failure Indicators: Tracking mechanism not added to Large PRD Protocol
    Evidence: .sisyphus/evidence/task-5-prd-tracking.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/blueprint/AGENTS.md`

- [ ] 6. [C2] Replace sed-based backfill with robust approach in DECOMPOSE

  **What to do**:
  - Open `agents/plan/decompose/AGENTS.md`
  - Replace the entire §2.2 backfill code block (lines ~188-205) with a robust approach:
    - Remove the `echo "$body" | sed 's/depends_on: pending/...'` pattern
    - Replace with a Python one-liner that targets only the `<!-- agent-meta -->` block:
      ```bash
      # For each issue that has dependencies:
      # 1. Get the current body
      body=$(gh issue view <issue_num> --repo <repo> --json body -q .body)
      
      # 2. Replace depends_on ONLY within the agent-meta block (not in prose)
      new_body=$(python3 -c "
      import re, sys
      body = sys.stdin.read()
      body = re.sub(
          r'(<!-- agent-meta.*?)depends_on: pending(.*?-->)',
          r'\1depends_on: #43,#44\2',
          body, flags=re.DOTALL
      )
      print(body)
      " <<< "$body")
      
      # 3. Update the issue
      gh issue edit <issue_num> --repo <repo> --body "$new_body"
      ```
    - Keep the "For issues with no dependencies" section but update it to match the same approach:
      ```
      For issues with no dependencies, replace `pending` with an empty value:
      depends_on:
      ```
  - Add a note after the code block: "**Why not sed?** The issue body is Markdown with code blocks and prose that may contain 'depends_on'. Targeting only the `<!-- agent-meta -->` block prevents accidental replacements."

  **Must NOT do**:
  - Do NOT change the two-pass strategy (Pass 1 → Pass 2 is correct)
  - Do NOT change the `gh issue view` or `gh issue edit` commands themselves
  - Do NOT introduce external dependencies beyond Python 3 stdlib

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: One section replacement in one file
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 7, 8, 9)
  - **Blocks**: Task 10
  - **Blocked By**: Task 1 (schema must be aligned first)

  **References**:

  **Pattern References**:
  - `agents/plan/decompose/AGENTS.md:185-205` — Current sed-based backfill to replace
  - `agents/plan/decompose/AGENTS.md:138-148` — agent-meta block format (regex target)

  **External References**:
  - Python `re.sub` with `re.DOTALL` — targets multi-line blocks safely

  **WHY Each Reference Matters**:
  - The current sed approach doesn't scope to the metadata block — the Python regex with DOTALL anchors to `<!-- agent-meta ... -->` boundaries

  **Acceptance Criteria**:
  - [ ] `grep -c "sed " agents/plan/decompose/AGENTS.md` → 0
  - [ ] `grep -c "python3 -c" agents/plan/decompose/AGENTS.md` → ≥1
  - [ ] `grep -c "agent-meta" agents/plan/decompose/AGENTS.md` → ≥2 (template + backfill)
  - [ ] "Why not sed?" note present

  **QA Scenarios**:

  ```
  Scenario: sed removed, Python approach in place
    Tool: Bash (grep)
    Preconditions: Task 6 edits applied
    Steps:
      1. grep -c "sed 's/depends_on" agents/plan/decompose/AGENTS.md (expect 0)
      2. grep -c "python3 -c" agents/plan/decompose/AGENTS.md (expect ≥1)
      3. grep -c "re.DOTALL" agents/plan/decompose/AGENTS.md (expect 1)
      4. grep -c "Why not sed" agents/plan/decompose/AGENTS.md (expect 1)
    Expected Result: Step 1 → 0, Steps 2-4 → ≥1
    Failure Indicators: sed still present or Python approach missing
    Evidence: .sisyphus/evidence/task-6-backfill-robust.txt

  Scenario: Two-pass strategy preserved
    Tool: Bash (grep)
    Preconditions: Task 6 edits applied
    Steps:
      1. grep -c "Pass 1" agents/plan/decompose/AGENTS.md (expect ≥2)
      2. grep -c "Pass 2" agents/plan/decompose/AGENTS.md (expect ≥2)
      3. grep -c "gh issue edit" agents/plan/decompose/AGENTS.md (expect ≥1)
    Expected Result: All return ≥ expected counts
    Failure Indicators: Pass structure was accidentally removed
    Evidence: .sisyphus/evidence/task-6-two-pass-preserved.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/plan/decompose/AGENTS.md`

- [ ] 7. [C3+H1] Add recovery rule + failure protocol to DECOMPOSE Rules

  **What to do**:
  - Open `agents/plan/decompose/AGENTS.md`
  - Add 3 new rules to the Rules section (after Rule 10, line ~252):

    ```
    11. **On retry after partial failure:** Before creating issues, check if task issues already exist for this plan 
        (search for issues with `plan: #<plan_issue>` in the body). If all issues exist with `depends_on: pending`, 
        skip Pass 1 entirely and run Pass 2 (backfill) only. If some issues exist, create only the missing ones, 
        then run Pass 2 for all.
    12. **Failure recovery by error type:**
        - **HTTP 403 (auth):** Do not retry. Report immediately.
        - **HTTP 422 (validation):** Fix the issue body and retry once.
        - **HTTP 429 (rate limit):** Wait 60 seconds and retry up to 3 times.
        - **HTTP 500/502/503 (server):** Wait 30 seconds and retry up to 2 times.
        - **Network timeout:** Retry once after 15 seconds.
        - **`gh` CLI not found:** Report immediately — cannot proceed.
        - After 3 consecutive failures of any type: STOP. Report all created issues + all failures in the JSON result block. Do not continue creating issues.
    13. **Validate wave ordering before posting plan:** No task in Wave N may depend on a task in Wave N+1 or later. 
        If detected, re-order waves to satisfy all dependencies before posting the plan comment.
    ```

  **Must NOT do**:
  - Do NOT renumber existing Rules 1-10
  - Do NOT modify existing rule text
  - Do NOT add rules unrelated to recovery/validation

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Appending rules to an existing section
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 6, 8, 9)
  - **Blocks**: Task 10
  - **Blocked By**: None (Rules section is independent of §2.2)

  **References**:

  **Pattern References**:
  - `agents/plan/decompose/AGENTS.md:241-252` — Existing Rules section to append to
  - `specs/archive/FEAT-plan-task-decomposition.md:385` — Recovery rule source: "On retry, check existing task issues for `depends_on: pending`"

  **WHY Each Reference Matters**:
  - Existing rules set the tone and specificity level — new rules must match
  - Archived spec already defined the recovery behavior — it just wasn't carried into the prompt

  **Acceptance Criteria**:
  - [ ] Rules section has ≥13 rules (was 10)
  - [ ] "retry after partial failure" rule present
  - [ ] HTTP error code table present (403, 422, 429, 500)
  - [ ] "Validate wave ordering" rule present
  - [ ] Existing Rules 1-10 unchanged

  **QA Scenarios**:

  ```
  Scenario: New rules appended correctly
    Tool: Bash (grep)
    Preconditions: Task 7 edits applied
    Steps:
      1. grep -c "On retry after partial failure" agents/plan/decompose/AGENTS.md (expect 1)
      2. grep -c "HTTP 403" agents/plan/decompose/AGENTS.md (expect 1)
      3. grep -c "HTTP 429" agents/plan/decompose/AGENTS.md (expect 1)
      4. grep -c "Validate wave ordering" agents/plan/decompose/AGENTS.md (expect 1)
      5. grep -c "^[0-9]*\." agents/plan/decompose/AGENTS.md (expect ≥13 — numbered rules)
    Expected Result: Steps 1-4 return 1 each, step 5 returns ≥13
    Failure Indicators: Rules missing or existing rules modified
    Evidence: .sisyphus/evidence/task-7-rules-added.txt

  Scenario: Existing rules preserved
    Tool: Bash (grep)
    Preconditions: Task 7 edits applied
    Steps:
      1. grep -c "Never create issues before Blueprint approves" agents/plan/decompose/AGENTS.md (expect 1)
      2. grep -c "JSON result block must be the last" agents/plan/decompose/AGENTS.md (expect 1)
    Expected Result: Both return 1 (Rules 1 and 10 intact)
    Failure Indicators: Existing rules overwritten
    Evidence: .sisyphus/evidence/task-7-rules-preserved.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/plan/decompose/AGENTS.md`

- [ ] 8. [H4+L1+L4] Wave validation + edge cases in DECOMPOSE

  **What to do**:
  - Open `agents/plan/decompose/AGENTS.md`
  - **H4 — Wave validation**: Already handled via Task 7 Rule 13. No separate edit needed — verify only.
  - **L1 — Zero P0 requirements edge case**: In §1.3 (line ~53, after rule 5: "Every P0 requirement must produce at least one task"), add:
    "If the spec has zero P0 requirements, flag this in the plan comment Notes section: 'Warning: No P0 requirements found. Confirm with Blueprint whether any requirements should be elevated to P0 before proceeding.' Do not block — proceed with P1/P2 task creation."
  - **L4 — Large task error reporting**: In §2.1 (line ~181, near "If Blueprint approved a task as `large`, something went wrong — do not create it; report the error"), replace with:
    "If Blueprint approved a task as `large`, something went wrong. Do not create it. Skip to the next task, and include the large task in the `issues_failed` array of the JSON result block with reason 'size: large — must be split before creation'. Continue creating remaining tasks."

  **Must NOT do**:
  - Do NOT modify §1.3 rules 1-4 or 6-7
  - Do NOT change the JSON result block format

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Two small insertions in non-overlapping sections
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 6, 7, 9)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `agents/plan/decompose/AGENTS.md:49-61` — §1.3 rules 1-7 to extend
  - `agents/plan/decompose/AGENTS.md:179-183` — Pass 1 rules to clarify
  - `agents/plan/decompose/AGENTS.md:226-237` — JSON error format to reference

  **WHY Each Reference Matters**:
  - L1 handles a vacuous truth: "every P0 must produce a task" passes when there are zero P0s — this could hide a spec quality problem
  - L4 clarifies what "report the error" means — use the existing JSON error format, don't invent a new one

  **Acceptance Criteria**:
  - [ ] "zero P0 requirements" or "No P0 requirements" warning text present in §1.3
  - [ ] "issues_failed" referenced for large task error reporting
  - [ ] Large task error includes reason string
  - [ ] Existing decomposition rules unchanged

  **QA Scenarios**:

  ```
  Scenario: Zero P0 edge case handled
    Tool: Bash (grep)
    Preconditions: Task 8 edits applied
    Steps:
      1. grep -c "No P0 requirements" agents/plan/decompose/AGENTS.md (expect 1)
      2. grep -c "elevated to P0" agents/plan/decompose/AGENTS.md (expect 1)
    Expected Result: Both return 1
    Failure Indicators: Edge case not documented
    Evidence: .sisyphus/evidence/task-8-zero-p0.txt

  Scenario: Large task error reporting clarified
    Tool: Bash (grep)
    Preconditions: Task 8 edits applied
    Steps:
      1. grep -c "issues_failed" agents/plan/decompose/AGENTS.md (expect ≥2 — in template and in rule)
      2. grep -c "must be split before creation" agents/plan/decompose/AGENTS.md (expect 1)
    Expected Result: Step 1 ≥2, step 2 = 1
    Failure Indicators: Error reporting mechanism unclear
    Evidence: .sisyphus/evidence/task-8-large-task-error.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/plan/decompose/AGENTS.md`

- [ ] 9. [L2+L3] Spec versioning + naming consistency

  **What to do**:
  - Open `agents/plan/decompose/AGENTS.md`
  - **L2 — Spec versioning in metadata**: In the issue body template (lines ~138-148), add a new field to the `<!-- agent-meta -->` block:
    ```
    spec_sha: <first 7 chars of git SHA for the spec file>
    ```
    Add after the `spec:` field. In §2.1 rules, add: "Obtain spec_sha by running `git log -1 --format='%h' -- <spec_path>` before creating issues."
  - **L3 — find-work naming consistency**: In `specs/FEAT-agent-workflow-system.md`, search for `find-work.sh` references in the PLAN agent section (around line ~362) and the example script (lines ~423-446). Add a note near the repo structure section (line ~362):
    ```
    > **Note:** PLAN uses `find-work.py` (Python) for richer JSON parsing and multi-repo logic. 
    > Other agents use `find-work.sh` (shell) for simpler single-query discovery.
    ```
    Do NOT rename the files — just document the intentional divergence.

  **Must NOT do**:
  - Do NOT change existing `spec:` field behavior
  - Do NOT rename any `find-work.*` references (they reflect actual planned implementations)
  - Do NOT modify the BUILD example `find-work.sh` script

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Two small additions across two files
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 6, 7, 8)
  - **Blocks**: Task 10
  - **Blocked By**: Task 1 (workflow spec must be updated first for L3)

  **References**:

  **Pattern References**:
  - `agents/plan/decompose/AGENTS.md:138-148` — Metadata template to extend
  - `specs/FEAT-agent-workflow-system.md:358-383` — Repo structure with find-work references
  - `specs/archive/FEAT-plan-task-decomposition.md:109-137` — find-work.py spec (confirms Python is intentional for PLAN)

  **WHY Each Reference Matters**:
  - The metadata template is the canonical issue format — adding `spec_sha` enables stale-task detection
  - The workflow spec's repo structure section is where the naming divergence is most visible

  **Acceptance Criteria**:
  - [ ] `spec_sha` field present in DECOMPOSE metadata template
  - [ ] `git log -1 --format='%h'` instruction present in §2.1
  - [ ] "find-work.py" vs "find-work.sh" note present in workflow spec
  - [ ] No `find-work.*` files were renamed

  **QA Scenarios**:

  ```
  Scenario: Spec versioning added to metadata
    Tool: Bash (grep)
    Preconditions: Task 9 edits applied
    Steps:
      1. grep -c "spec_sha:" agents/plan/decompose/AGENTS.md (expect ≥1)
      2. grep -c "git log -1" agents/plan/decompose/AGENTS.md (expect 1)
    Expected Result: Both return ≥1
    Failure Indicators: spec_sha field or git command missing
    Evidence: .sisyphus/evidence/task-9-spec-versioning.txt

  Scenario: Naming divergence documented
    Tool: Bash (grep)
    Preconditions: Task 9 edits applied
    Steps:
      1. grep -c "find-work.py" specs/FEAT-agent-workflow-system.md (expect ≥1)
      2. grep -c "find-work.sh" specs/FEAT-agent-workflow-system.md (expect ≥1 — preserved)
      3. grep -c "richer JSON parsing" specs/FEAT-agent-workflow-system.md (expect 1)
    Expected Result: All return ≥1
    Failure Indicators: Note missing or files renamed
    Evidence: .sisyphus/evidence/task-9-naming-consistency.txt
  ```

  **Commit**: YES (groups with all tasks)
  - Files: `agents/plan/decompose/AGENTS.md`, `specs/FEAT-agent-workflow-system.md`

- [ ] 10. Cross-file consistency verification

  **What to do**:
  - After all Wave 1 and Wave 2 tasks are complete, run a cross-file consistency check:
  - **Schema alignment**: Verify `<!-- agent-meta -->` blocks in workflow spec and DECOMPOSE have identical field names (not values — templates use different example values)
  - **Heading hierarchy**: Read each of the 3 files end-to-end. Verify no orphaned headings (e.g., H4 under H2 with no H3)
  - **Cross-references**: Verify Blueprint references to Phase numbers still match (Phase 1-8)
  - **find-work naming**: Verify the new note in workflow spec accurately describes the py/sh split
  - **Rule numbering**: Verify DECOMPOSE rules are numbered 1-13 sequentially with no gaps
  - **Blueprint line count**: Verify ~540-560 lines (was 509 + ~30-50 lines of additions)
  - **DECOMPOSE line count**: Verify ~290-310 lines (was 252 + ~40-60 lines of additions)

  **Must NOT do**:
  - Do NOT make any edits in this task — verification only
  - If issues found, report them for manual resolution

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Read + grep verification, no edits
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3 (solo)
  - **Blocks**: F1-F4
  - **Blocked By**: Tasks 2, 3, 4, 5, 6, 7, 8, 9

  **References**:

  **Pattern References**:
  - All 3 target files post-edit

  **WHY Each Reference Matters**:
  - This is the integration check — ensures 9 parallel edits didn't create inconsistencies

  **Acceptance Criteria**:
  - [ ] Schema field names match between workflow spec and DECOMPOSE
  - [ ] No heading hierarchy violations in any file
  - [ ] Blueprint phases still numbered 1-8
  - [ ] DECOMPOSE rules numbered 1-13
  - [ ] Line counts within expected ranges

  **QA Scenarios**:

  ```
  Scenario: Metadata fields consistent across files
    Tool: Bash (grep + diff)
    Preconditions: All Wave 1+2 tasks complete
    Steps:
      1. Extract field names from workflow spec metadata block
      2. Extract field names from DECOMPOSE metadata template
      3. Compare — field names must be identical (order may differ)
    Expected Result: Same fields in both files: attempts_build, attempts_test, attempts_review, max_attempts, size, feature_branch, spec, spec_sha, plan, pr, depends_on
    Failure Indicators: Field name mismatch between files
    Evidence: .sisyphus/evidence/task-10-schema-consistency.txt

  Scenario: Heading hierarchy valid
    Tool: Bash (grep)
    Preconditions: All Wave 1+2 tasks complete
    Steps:
      1. grep -n "^#" agents/blueprint/AGENTS.md | verify no H4 appears without preceding H3
      2. grep -n "^#" agents/plan/decompose/AGENTS.md | same check
      3. grep -n "^#" specs/FEAT-agent-workflow-system.md | same check
    Expected Result: All files have valid heading hierarchies
    Failure Indicators: Heading level skip (e.g., ## followed by ####)
    Evidence: .sisyphus/evidence/task-10-heading-hierarchy.txt

  Scenario: Rule numbering sequential
    Tool: Bash (grep)
    Preconditions: All Wave 1+2 tasks complete
    Steps:
      1. grep -n "^[0-9]*\. \*\*" agents/plan/decompose/AGENTS.md
      2. Verify numbers are 1, 2, 3, ..., 13 with no gaps or duplicates
    Expected Result: Rules numbered 1-13 sequentially
    Failure Indicators: Gap, duplicate, or misnumbered rule
    Evidence: .sisyphus/evidence/task-10-rule-numbering.txt
  ```

  **Commit**: NO (verification only)

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `unspecified-high`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (grep for field names, read sections). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Markdown Quality + Rendering Check** — `unspecified-high`
  Read all 3 changed files end-to-end. Verify: heading hierarchy (no orphan H4 under H2), code block fences matched, table alignment, no broken links/references. Check line counts match expected ranges. Flag any formatting issues.
  Output: `Files [3/3 clean] | Headings [OK/N issues] | Code blocks [OK/N issues] | VERDICT`

- [ ] F3. **Schema Consistency Grep** — `quick`
  Run: `grep -rn "attempts:" specs/ agents/` and verify: (a) no standalone `attempts: 0` remains (only `attempts_build/test/review`), (b) `depends_on` field name consistent, (c) `max_attempts: 3` consistent, (d) no `sed` commands in DECOMPOSE. Cross-check all metadata blocks match the canonical schema.
  Output: `Old schema refs [0] | New schema refs [N] | sed refs [0] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual changes. Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance. Verify no changes to: archived specs, SETUP.md, reference files. Flag any unaccounted edits.
  Output: `Tasks [N/N compliant] | Forbidden files [CLEAN] | Scope [CLEAN/N issues] | VERDICT`

---

## Commit Strategy

- **Single commit** after all fixes: `fix(agents): resolve 15 review issues in PR #16 prompt improvements`
  - Files: `agents/blueprint/AGENTS.md`, `agents/plan/decompose/AGENTS.md`, `specs/FEAT-agent-workflow-system.md`
  - Pre-commit: `grep -c "attempts: 0" specs/FEAT-agent-workflow-system.md` → expect 0

---

## Success Criteria

### Verification Commands
```bash
# Schema alignment
grep -c "attempts: 0" specs/FEAT-agent-workflow-system.md  # Expected: 0
grep -c "attempts_build: 0" specs/FEAT-agent-workflow-system.md  # Expected: 1
grep -c "attempts_build: 0" agents/plan/decompose/AGENTS.md  # Expected: 1 (in template)

# No sed in DECOMPOSE
grep -c "sed " agents/plan/decompose/AGENTS.md  # Expected: 0

# Line counts (approximate)
wc -l agents/blueprint/AGENTS.md  # Expected: ~540-560
wc -l agents/plan/decompose/AGENTS.md  # Expected: ~290-310
wc -l specs/FEAT-agent-workflow-system.md  # Expected: ~500-510
```

### Final Checklist
- [ ] All "Must Have" present
- [ ] All "Must NOT Have" absent
- [ ] Three-counter schema consistent across all files
- [ ] No raw sed in DECOMPOSE
- [ ] Backfill recovery rule present
- [ ] Failure recovery protocol present
- [ ] Wave validation rule present
- [ ] All markdown renders cleanly
