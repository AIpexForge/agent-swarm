# PRD: PLAN Phase B — Task Decomposition

**Author:** Blueprint (AI) + George Sapp
**Date:** 2026-03-01
**Status:** Draft
**Version:** 1.1
**Target Repo:** AIpexForge/agent-swarm
**Taskmaster Optimized:** No — uses native LLM decomposition

---

## Changes from v1.0

- **REQ-007 (Scope Review) reinstated for v1** — Simplified from full LLM estimation to single-pass classification: small (<15m), medium (15-30m), large (>30m). Split tasks >60m, merge tightly-coupled <10m tasks. Rules-based with LLM judgment on sizing — not a separate estimation pipeline.
- **REQ-008 simplified** — Removed 50% threshold heuristic. Always complete remaining tasks on partial failure (never delete). Task plan persisted to plan issue comment before issue creation begins.
- **REQ-008 duplicate detection** — Switched from title-based search to hash-based ID (`decomp_id` in metadata block) for deterministic matching.
- **REQ-002 retry** — Retry path now runs duplicate detection (REQ-008) before re-invoking sub-agent.
- **REQ-009 two-pass backfill** — Explicitly documented: Pass 1 creates all issues with `depends_on: pending`, Pass 2 backfills with real issue numbers via `gh issue edit`.
- **Dependency chain corrected** — REQ-004 (feature branch) now precedes REQ-003 (task issue structure), matching the architecture diagram.
- **Sub-agent output format aligned** — REQ-006 and Technical Considerations now both specify the full JSON object with `issues_created` and `dependency_map`.
- **Cron → Blueprint session spawn** — Documented explicitly in Technical Considerations.
- **Open Questions resolved** — Q1 (remote detection: `git remote get-url origin`), Q2 (`decomposed` added alongside `plan:draft`), Q4 (sub-agent timeout: 10 minutes).
- **testStrategies hardened** — Added concrete `gh` CLI commands and assertion patterns to P0 requirements.
- **Label bootstrap** — REQ-005 now ensures `ready-for-build` and `decomposed` labels exist before decomposition begins.
- **Prompt file path** — Changed to `agents/plan/decompose/AGENTS.md` for consistency with repo conventions.

---

## Executive Summary

PLAN Phase B is Blueprint's cron-driven capability that detects merged spec PRs, decomposes them into task issues sized for ~30 minute BUILD agent execution, and creates a feature branch linking all tasks. It bridges the gap between "spec approved" and "BUILD has work to do." Without it, the pipeline stalls after planning — specs exist but no actionable tasks flow to BUILD.

## Problem Statement

### Current Situation
Blueprint can produce comprehensive PRD specs via interactive planning (Phase A, PR #10) and onboard repos (ONBOARD, PR #12). However, after a spec PR is merged, nothing happens. There is no automated process to break specs into task issues that BUILD can pick up.

### User Impact
George must manually create task issues from merged specs — reading the spec, deciding task boundaries, writing issue bodies with metadata blocks, creating feature branches, and labeling everything correctly. This is tedious, error-prone, and defeats the purpose of an autonomous agent pipeline.

### Business Impact
Task decomposition is the critical bridge in the pipeline. BUILD, TEST, REVIEW, and MERGE are all downstream of it. Until Phase B exists, the entire pipeline after PLAN is manual.

### Why Now
PLAN Phase A (interactive mode) and ONBOARD are complete. Phase B is the next link in the build order. BUILD agent development is blocked on having task issues to consume.

---

## Goals & Success Metrics

| Goal | Metric | Baseline (source) | Target | Timeframe | Measurement |
|------|--------|--------------------|--------|-----------|-------------|
| Automated decomposition | % of merged specs decomposed without human intervention | 0% (manual) | ≥90% | Per spec | Count of specs requiring manual decomposition |
| Appropriate task sizing | % of tasks completable by BUILD in ≤30 min | N/A | ≥80% | Per decomposition | BUILD agent execution time logs |
| Correct metadata | % of task issues with valid metadata blocks + labels | N/A | 100% | Per decomposition | Automated metadata validation |
| Zero-cost idle | Token spend when no merged specs exist | N/A | $0 | Per cron run | find-work.py exits before LLM invocation |
| Pipeline throughput | Time from spec merge to task issues created | ∞ (manual) | ≤30 min (next cron) | Per spec | Timestamp diff: merge time vs first task issue creation |

---

## User Stories

### US-001: Automatic Task Issue Creation from Merged Spec
- **As a** developer using agent-swarm
- **I want** merged spec PRs to automatically produce task issues
- **So that** BUILD can start working without manual issue creation
- **Acceptance Criteria:**
  1. Within one cron cycle after a spec PR is merged, task issues appear on the target repo
  2. Each task issue has a valid metadata block, acceptance criteria, testStrategy, and spec link
  3. Each task issue is labeled `ready-for-build`
  4. A feature branch `feat/<plan-slug>` exists for task PRs to target
  5. The plan issue is updated with a comment listing all created task issues
  6. The spec PR is labeled `decomposed` (alongside existing `plan:draft`)
- **Task Hints:** find-work.py (2h), decomposition sub-agent prompt (3h), issue creation logic (2h), feature branch creation (30m)
- **Dependencies:** REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006

### US-002: Idempotent Decomposition
- **As a** developer
- **I want** the cron job to safely skip specs that have already been decomposed
- **So that** duplicate task issues are never created
- **Acceptance Criteria:**
  1. Specs with the `decomposed` label on their PR are skipped
  2. If decomposition fails mid-way (some issues created, some not), the next cron run detects partial state and completes remaining tasks
  3. No duplicate task issues are created under any circumstances
- **Task Hints:** Idempotency detection (1h), partial state recovery (1.5h)
- **Dependencies:** REQ-001, REQ-008

---

## Functional Requirements

### P0 — Must Have

#### REQ-001: Work Discovery Script (`find-work.py`)
- **Description:** A Python script that runs before any LLM invocation, querying GitHub for merged spec PRs that need decomposition. If no work exists, it exits immediately with zero token spend.
- **Acceptance Criteria:**
  1. Script is located at `agents/plan/scripts/find-work.py`
  2. Scans all repos present in `~/.openclaw/workspace/` that have `.agents/commands.yml`
  3. Resolves GitHub remote via `git -C <repo_path> remote get-url origin`, parsing the owner/repo from the URL. If remote resolution fails, logs a warning and skips the repo.
  4. For each repo, queries: `gh pr list --state merged --label plan:draft --json number,title,files,mergedAt,labels`
  5. Filters out PRs that have the `decomposed` label (both `plan:draft` AND `decomposed` present = already processed)
  6. `plan:draft` label is preserved on merged PRs — it is never removed by any agent. `decomposed` is added alongside it.
  7. If no qualifying PRs found: exits with code 1 (no LLM session spawned)
  8. If qualifying PRs found: exits with code 0, outputs JSON to stdout with PR number, repo, spec file path, and plan issue number
  9. Output format:
     ```json
     [
       {
         "repo": "AIpexForge/snaphappy",
         "repo_path": "/home/gsapp/.openclaw/workspace/snaphappy",
         "pr_number": 42,
         "spec_path": "specs/FEAT-add-webhook-handler.md",
         "plan_issue": 41,
         "merged_at": "2026-03-01T12:00:00Z"
       }
     ]
     ```
  10. Processes only the oldest qualifying PR (sequential v1 — one spec at a time)
- **testStrategy:**
  - Acceptance criteria: Script correctly identifies merged `plan:draft` PRs without `decomposed` label, outputs valid JSON, exits 1 when no work
  - Verification approach:
    ```bash
    # Positive case: merged PR with plan:draft, no decomposed
    python3 agents/plan/scripts/find-work.py
    # Assert: exit code 0, stdout is valid JSON array with expected fields
    echo $? # should be 0
    echo "$output" | python3 -c "import json,sys; d=json.load(sys.stdin); assert len(d)>=1; assert all(k in d[0] for k in ['repo','repo_path','pr_number','spec_path','plan_issue','merged_at'])"

    # Negative case: add decomposed label, re-run
    gh pr edit $PR_NUM --repo $REPO --add-label decomposed
    python3 agents/plan/scripts/find-work.py
    # Assert: exit code 1

    # Skip case: repo without .agents/commands.yml
    # Assert: repo not in output
    ```
  - Edge cases to test: Multiple merged spec PRs (only oldest processed), PR with both `plan:draft` and `decomposed` (skipped), repo without `.agents/commands.yml` (skipped), repo with renamed remote (warning logged, skipped)
  - Performance threshold: Script completes in under 5 seconds for up to 10 repos
- **Task Breakdown:** Repo discovery + remote resolution (30m), GitHub query logic (30m), JSON output formatting (30m), exit code logic (15m), testing (15m)
- **Dependencies:** None

#### REQ-004: Feature Branch Creation
- **Description:** Blueprint creates a feature branch on the target repo during decomposition, before any task issues are created. All task PRs will target this branch.
- **Acceptance Criteria:**
  1. Branch name: `feat/<plan-slug>` (e.g., `feat/add-webhook-handler`)
  2. Branch is created from the repo's default branch (detected via `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`)
  3. Branch is pushed to the remote: `git push origin feat/<plan-slug>`
  4. If branch already exists (from a previous partial decomposition), it is reused — not deleted and recreated
  5. All task issue metadata blocks reference this branch in the `feature_branch` field
  6. On escalation (decomposition fails after retry), Blueprint logs a comment on the spec PR noting the branch was created but decomposition failed
- **testStrategy:**
  - Acceptance criteria: Branch exists on remote, is based on default branch, task metadata references it
  - Verification approach:
    ```bash
    # Verify branch exists
    git ls-remote --heads origin feat/<slug> | grep -q feat/<slug>
    # Verify based on default branch
    DEFAULT=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
    git log --oneline feat/<slug>..$DEFAULT | wc -l  # should be 0 (no divergence)
    # Verify task metadata
    gh issue list --repo <target> --label ready-for-build --json body \
      | python3 -c "import json,sys; issues=json.load(sys.stdin); assert all('feature_branch: feat/<slug>' in i['body'] for i in issues)"
    ```
  - Edge cases to test: Branch already exists (reuse), default branch is not `main`, escalation path (comment on spec PR)
- **Task Breakdown:** Branch creation logic (15m), default branch detection (15m), idempotency check (15m), escalation cleanup (15m)
- **Dependencies:** REQ-001

#### REQ-002: Decomposition Sub-Agent Spawning
- **Description:** Blueprint spawns a focused sub-agent for each spec that needs decomposition. The sub-agent receives the spec content, codebase context, and a decomposition prompt, then creates task issues directly via `gh issue create`.
- **Acceptance Criteria:**
  1. Blueprint reads the merged spec file and relevant codebase context (AGENTS.md, commands.yml, directory structure)
  2. Blueprint spawns a sub-agent with: spec content, codebase context, decomposition prompt, target repo, feature branch name, plan issue number
  3. The sub-agent creates task issues directly using `gh issue create` (no intermediate file handoff)
  4. The sub-agent returns a JSON block in its final message with created issue numbers and dependency map
  5. If the sub-agent fails or returns malformed output, Blueprint runs duplicate detection (REQ-008) then retries once with the same inputs
  6. If retry fails, Blueprint labels the spec PR `escalated` and notifies George via Telegram
  7. Sub-agent timeout: 10 minutes. If exceeded, Blueprint treats it as a failure and follows the retry path.
- **testStrategy:**
  - Acceptance criteria: Sub-agent receives correct context, creates valid issues, returns issue number list
  - Verification approach:
    ```bash
    # After decomposition, verify issues exist
    gh issue list --repo <target> --label ready-for-build --json number,title,body,labels

    # Verify each issue has valid metadata block
    for num in $ISSUE_NUMBERS; do
      body=$(gh issue view $num --repo <target> --json body -q .body)
      echo "$body" | grep -q '<!-- agent-meta'
      echo "$body" | grep -q 'attempts_build: 0'
      echo "$body" | grep -q 'feature_branch:'
      echo "$body" | grep -q 'spec:'
      echo "$body" | grep -q 'plan:'
      echo "$body" | grep -q 'decomp_id:'
    done
    ```
  - Edge cases to test: Sub-agent fails mid-creation (partial issues exist), sub-agent returns empty list, sub-agent times out at 10 min
- **Task Breakdown:** Context assembly (1h), sub-agent prompt (2h), spawn + result handling (1h), retry logic with duplicate detection (30m)
- **Dependencies:** REQ-001, REQ-004, REQ-008

#### REQ-003: Task Issue Structure
- **Description:** Each task issue created by the decomposition sub-agent follows a consistent structure with metadata block, description, acceptance criteria, testStrategy, and context links.
- **Acceptance Criteria:**
  1. Issue title format: `[TASK] <concise task description>`
  2. Issue body contains metadata block as hidden HTML comment:
     ```markdown
     <!-- agent-meta
     attempts_build: 0
     attempts_test: 0
     attempts_review: 0
     max_attempts: 3
     feature_branch: feat/<plan-slug>
     spec: specs/FEAT-<name>.md
     plan: #<plan-issue-number>
     pr:
     depends_on: #<issue>,#<issue>
     decomp_id: <sha256-hash>
     -->
     ```
  3. `decomp_id` is a deterministic hash of `spec_path + requirement_id + task_index` (e.g., SHA256 of `specs/FEAT-foo.md:REQ-003:2`). Used for duplicate detection.
  4. Issue body contains after metadata block:
     - **Description:** What this task accomplishes (2-5 sentences)
     - **Acceptance Criteria:** Numbered list pulled/derived from the spec's REQ acceptance criteria
     - **Test Strategy:** How BUILD's output will be validated, pulled from the spec's testStrategy
     - **Spec Reference:** Link to the spec file in the repo (e.g., `[spec](../../specs/FEAT-<name>.md)`)
     - **Files Likely Affected:** List of files/directories the task will probably touch (inferred from codebase scan)
     - **Dependencies:** Human-readable list of prerequisite task issues with links
  5. Issue is labeled `ready-for-build`
  6. Each task targets a single focused concern completable in ~30 min agent execution time
- **testStrategy:**
  - Acceptance criteria: Every created issue matches the structure above, metadata is parseable, all links resolve
  - Verification approach:
    ```bash
    # For each task issue:
    body=$(gh issue view $NUM --repo <target> --json body -q .body)

    # Validate metadata block fields
    echo "$body" | python3 -c "
    import sys, re, hashlib
    body = sys.stdin.read()
    meta = re.search(r'<!-- agent-meta\n(.*?)\n-->', body, re.DOTALL)
    assert meta, 'No metadata block'
    fields = dict(line.split(': ', 1) for line in meta.group(1).strip().split('\n'))
    for key in ['attempts_build','attempts_test','attempts_review','max_attempts','feature_branch','spec','plan','decomp_id']:
        assert key in fields, f'Missing field: {key}'
    "

    # Validate body sections
    for section in 'Description' 'Acceptance Criteria' 'Test Strategy' 'Spec Reference' 'Files Likely Affected'; do
      echo "$body" | grep -q "## $section\|### $section\|**$section"
    done

    # Validate label
    gh issue view $NUM --repo <target> --json labels -q '.labels[].name' | grep -q 'ready-for-build'
    ```
  - Edge cases to test: Task with no dependencies (depends_on is empty), task referencing a file that doesn't exist yet (acceptable), spec with only 1 requirement (produces 1-2 tasks)
- **Task Breakdown:** Issue body template (1h), metadata block generation with hash (30m), acceptance criteria extraction (1h), file inference (30m)
- **Dependencies:** REQ-004, REQ-006

#### REQ-005: Plan Issue Update and Label Management
- **Description:** After decomposition, Blueprint adds a summary comment to the plan issue listing all created task issues and labels the spec PR `decomposed`. Also ensures required labels exist before decomposition begins.
- **Acceptance Criteria:**
  1. Before decomposition begins, Blueprint verifies `ready-for-build` and `decomposed` labels exist on the target repo. If missing, creates them via `gh label create --force`.
  2. Comment on the plan issue contains:
     - Number of tasks created
     - Numbered list of task issues with titles and links
     - Dependency graph (simple text: "Task #N depends on #M")
     - Feature branch name
  3. Comment uses the standard agent comment format:
     ```markdown
     <details>
     <summary>📐 PLAN — Decomposed spec into N tasks (Xm Ys)</summary>

     **Feature branch:** `feat/<slug>`
     **Tasks created:**
     1. #43 — Implement user auth endpoint
     2. #44 — Add validation middleware (depends on #43)
     ...

     </details>
     ```
  4. The spec PR receives the `decomposed` label (added alongside existing `plan:draft`)
- **testStrategy:**
  - Acceptance criteria: Plan issue has summary comment with all required fields, spec PR has `decomposed` label
  - Verification approach:
    ```bash
    # Verify comment on plan issue
    comments=$(gh issue view <plan> --repo <target> --json comments -q '.comments[-1].body')
    echo "$comments" | grep -q 'PLAN — Decomposed spec into'
    echo "$comments" | grep -q 'feat/'
    echo "$comments" | grep -qP '#\d+'  # at least one issue link

    # Verify decomposed label on spec PR
    gh pr view <spec-pr> --repo <target> --json labels -q '.labels[].name' | grep -q 'decomposed'

    # Verify plan:draft still present
    gh pr view <spec-pr> --repo <target> --json labels -q '.labels[].name' | grep -q 'plan:draft'
    ```
  - Edge cases to test: Plan issue already has comments from Phase A (new comment appends), labels don't exist yet (auto-created)
- **Task Breakdown:** Label bootstrap (15m), comment template (30m), PR label update (15m)
- **Dependencies:** REQ-002, REQ-003

#### REQ-006: Decomposition Prompt
- **Description:** The prompt that drives the decomposition sub-agent. It instructs the LLM to read a spec, understand the codebase, and create appropriately-scoped task issues directly via `gh issue create`.
- **Acceptance Criteria:**
  1. Prompt file exists at `agents/plan/decompose/AGENTS.md`
  2. Prompt includes:
     - Role: "You are a task decomposition agent"
     - Input specification: spec content, codebase context, target repo, feature branch, plan issue number
     - Task sizing guidance: each task should be completable by BUILD in ~30 minutes (single focused concern, ~200 lines changed max)
     - Issue structure template (matching REQ-003 exactly, including `decomp_id` hash generation)
     - Two-pass dependency strategy: Pass 1 — create all issues with `depends_on: pending`. Pass 2 — backfill `depends_on` with real issue numbers via `gh issue edit`.
     - Instruction to create issues directly via `gh issue create`
     - Final output: a JSON block with the full result object:
       ```json
       {
         "issues_created": [43, 44, 45, 46, 47],
         "dependency_map": {
           "44": [43],
           "45": [43, 44],
           "46": [45],
           "47": [45, 46]
         }
       }
       ```
  3. Prompt is self-contained — the sub-agent can execute with only this prompt + the provided context
  4. Prompt explicitly instructs: derive acceptance criteria and testStrategy from the spec's REQ sections, not invented
  5. Prompt instructs: before creating any issues, post a "decomposition plan" comment on the plan issue listing intended tasks (titles + order). This persists the task plan for partial recovery (REQ-008).
- **testStrategy:**
  - Acceptance criteria: Prompt file exists, contains all required sections, sub-agent can execute from it
  - Verification approach: Load prompt in a test session with the ONBOARD spec (`specs/FEAT-onboard-capability.md`) against the agent-swarm repo. Verify: (a) decomposition plan comment appears on plan issue before task creation, (b) task issues are created with valid structure per REQ-003, (c) `depends_on` fields are backfilled with real issue numbers after all issues exist, (d) final message contains valid JSON with `issues_created` and `dependency_map`
  - Edge cases to test: Spec with only P1 requirements (no P0), spec with complex dependency chains between requirements
- **Task Breakdown:** Prompt writing (2h), testing and iteration (1h)
- **Dependencies:** REQ-003

#### REQ-008: Idempotency and Partial State Recovery
- **Description:** The decomposition process must be idempotent — running it twice on the same spec must not create duplicate issues. If decomposition fails partway through, the next run must detect and handle the partial state.
- **Acceptance Criteria:**
  1. `find-work.py` skips specs with the `decomposed` label (primary idempotency gate)
  2. Before creating issues, the sub-agent posts a decomposition plan comment on the plan issue (see REQ-006 AC5). This persists the intended task list for recovery.
  3. If decomposition fails after creating some issues but before adding the `decomposed` label:
     - Next cron run detects existing task issues for this spec by querying for issues containing the spec's `decomp_id` prefix in their metadata block
     - Blueprint reads the decomposition plan comment to determine the full intended task list
     - Blueprint completes only the missing tasks (never deletes existing ones)
     - Once all tasks exist, adds `decomposed` label
  4. Duplicate detection uses `decomp_id` (deterministic hash per REQ-003 AC3): before creating an issue, check if an issue with the same `decomp_id` already exists:
     ```bash
     gh issue list --repo <target> --search "decomp_id: <hash>" --json number
     ```
  5. All state transitions (label additions, issue creation) are logged as comments for debugging
- **testStrategy:**
  - Acceptance criteria: No duplicate issues created on re-run, partial state is detected and completed
  - Verification approach:
    ```bash
    # Test 1: Full idempotency — run twice
    # First run: decomposition completes, decomposed label added
    # Second run: find-work.py exits 1 (no work)
    python3 agents/plan/scripts/find-work.py; echo "exit: $?"  # should be 1

    # Test 2: Partial failure recovery
    # Simulate: create 2 of 5 tasks manually (with correct decomp_ids), don't add decomposed label
    # Run decomposition: should create only the 3 missing tasks
    gh issue list --repo <target> --label ready-for-build --json number | python3 -c "import json,sys; assert len(json.load(sys.stdin)) == 5"

    # Test 3: Duplicate prevention
    # Try to create an issue with an existing decomp_id
    # Assert: issue is skipped, not duplicated
    ```
  - Edge cases to test: Decomposition fails after creating 0 issues (clean re-run), fails after all issues but before labeling (just add label), concurrent repo activity creating issues between tasks
- **Task Breakdown:** Hash-based duplicate detection (30m), decomposition plan persistence (30m), partial state detection + completion (1h), logging (30m)
- **Dependencies:** REQ-001, REQ-003, REQ-005

### P1 — Should Have

#### REQ-009: Dependency-Aware Task Ordering
- **Description:** Tasks are created in dependency order so that prerequisite tasks have lower issue numbers. The `depends_on` field references real GitHub issue numbers, backfilled after all issues exist.
- **Acceptance Criteria:**
  1. Pass 1: Sub-agent creates all tasks in logical dependency order (foundations first), with `depends_on: pending` in metadata
  2. Pass 2: After all issues are created, sub-agent updates each issue's metadata block via `gh issue edit`, replacing `pending` with actual issue number references
  3. No circular dependencies exist in the created task set
  4. BUILD can use `depends_on` to skip tasks whose prerequisites aren't labeled `complete`
- **testStrategy:**
  - Acceptance criteria: Task issues are numbered in dependency order, `depends_on` references are valid after backfill
  - Verification approach:
    ```bash
    # Fetch all task issues, build dependency graph
    issues=$(gh issue list --repo <target> --label ready-for-build --json number,body)
    python3 -c "
    import json, sys, re
    issues = json.loads(sys.stdin.read())
    graph = {}
    for issue in issues:
        meta = re.search(r'depends_on: (.*)', issue['body'])
        deps = []
        if meta and meta.group(1).strip() and meta.group(1).strip() != 'pending':
            deps = [int(x.strip().lstrip('#')) for x in meta.group(1).split(',')]
        graph[issue['number']] = deps
    # Verify DAG (no cycles)
    visited, stack = set(), set()
    def has_cycle(n):
        if n in stack: return True
        if n in visited: return False
        stack.add(n); visited.add(n)
        for dep in graph.get(n, []):
            if has_cycle(dep): return True
        stack.discard(n); return False
    assert not any(has_cycle(n) for n in graph), 'Circular dependency detected'
    # Verify deps have lower issue numbers
    for num, deps in graph.items():
        for dep in deps:
            assert dep < num, f'Issue #{num} depends on #{dep} which has a higher number'
    " <<< "$issues"
    ```
  - Edge cases to test: Spec where all tasks are independent (no dependencies), spec with deep chain (A → B → C → D), concurrent issue creation on repo causing non-sequential numbers (still valid if `depends_on` references correct numbers)
- **Task Breakdown:** Two-pass logic in sub-agent prompt (30m), `gh issue edit` backfill (30m), DAG validation (30m)
- **Dependencies:** REQ-003, REQ-006

#### REQ-010: Telegram Notification on Decomposition
- **Description:** After decomposition completes, Blueprint notifies George via Telegram with a summary.
- **Acceptance Criteria:**
  1. Message includes: spec name, number of tasks created, feature branch name, plan issue link
  2. Message is concise (under 15 lines)
  3. On escalation (decomposition failed after retry), message includes error context and link to the spec PR
- **testStrategy:**
  - Acceptance criteria: Notification contains spec name, task count, branch name, plan issue link
  - Verification approach: Manual verification — after test decomposition, confirm Telegram message received with all required fields. Automated: mock the notification call in test, assert message string matches pattern `/<spec-name>.*\d+ tasks.*feat\/.*#\d+/`
- **Task Breakdown:** Message template (15m), integration (15m)
- **Dependencies:** REQ-005

#### REQ-007: Post-Decomposition Scope Review
- **Description:** After the decomposition sub-agent creates task issues, Blueprint performs a single-pass scope review. Each task is classified by estimated BUILD agent execution time: small (<15 min), medium (15-30 min), or large (>30 min). Tasks estimated at 60+ min are split; tightly-coupled tasks under 10 min each are merged. Target: every task completable in ~30 min agent time.
- **Acceptance Criteria:**
  1. Blueprint reviews all created task issues after decomposition completes (before adding `decomposed` label)
  2. Each task is classified as small / medium / large based on estimated agent minutes
  3. Tasks classified as large (>60 min estimated) are split into smaller issues. Original issue is updated to reference the new sub-issues, and the new issues receive correct metadata blocks and `ready-for-build` labels.
  4. Tightly-coupled small tasks (<10 min each, sequential dependency, touching the same files) may be merged into a single issue. The merged issue inherits the union of acceptance criteria and the stricter dependency set.
  5. Split/merge operations update the dependency chain — no orphaned or circular references after modification
  6. The plan issue summary comment (REQ-005) reflects the final task list after scope review (post-split/merge)
  7. If no tasks need splitting or merging, scope review is a no-op — no issues are modified
- **testStrategy:**
  - Acceptance criteria: After scope review, no task is estimated at >60 min, dependency chain is valid, plan comment reflects final state
  - Verification approach:
    ```bash
    # Verify plan issue comment lists final task count (post-split/merge)
    comment=$(gh issue view <plan> --repo <target> --json comments -q '.comments[-1].body')
    echo "$comment" | grep -q 'PLAN — Decomposed spec into'

    # Verify no orphaned dependencies
    issues=$(gh issue list --repo <target> --label ready-for-build --json number,body)
    python3 -c "
    import json, sys, re
    issues = json.loads(sys.stdin.read())
    nums = {i['number'] for i in issues}
    for i in issues:
        meta = re.search(r'depends_on: (.*)', i['body'])
        if meta and meta.group(1).strip() not in ('', 'pending'):
            deps = [int(x.strip().lstrip('#')) for x in meta.group(1).split(',')]
            for d in deps:
                assert d in nums, f'Issue #{i["number"]} depends on #{d} which does not exist'
    " <<< "$issues"

    # Verify dependency DAG is acyclic (reuse REQ-009 validation)
    ```
  - Edge cases to test: All tasks are medium (no-op), one task needs splitting (new issues created with correct metadata), two small tasks merged (original issues updated), split creates a new dependency chain
- **Task Breakdown:** Scope classification prompt (30m), split logic + issue creation (1h), merge logic + issue update (1h), dependency chain repair (30m)
- **Dependencies:** REQ-002, REQ-003, REQ-005
---

## Non-Functional Requirements

- **Performance:** `find-work.py` completes in under 5 seconds for up to 10 repos. Full decomposition (sub-agent spawn, issue creation, backfill) completes in under 10 minutes per spec.
- **Cost efficiency:** Zero token spend when no work exists. `find-work.py` uses only `gh` CLI calls (GitHub API), no LLM.
- **Reliability:** Idempotent — safe to run repeatedly. Partial failures are detected and completed on next run. No duplicate issues under any circumstances (hash-based detection).
- **Security:** Sub-agent has access only to `gh` CLI and file read tools. Does not execute code from the target repo.
- **Prerequisites:** Target repo must be onboarded (`.agents/commands.yml` exists). `gh` CLI authenticated with push access. Labels auto-created by REQ-005 if missing. `plan:draft` label exists (created by ONBOARD).

---

## Technical Considerations

### System Architecture

```
Cron fires (every 30 min at :00)
  │
  ▼
find-work.py
  │
  ├─ No work → exit 1 (zero cost)
  │
  └─ Work found → exit 0, output JSON to stdout
       │
       ▼
Cron wrapper spawns Blueprint session:
  openclaw session run --agent blueprint \
    --task "Decompose spec: <JSON from find-work.py>"
  │
  ├─ Concurrency guard: if a Blueprint decomposition session
  │  is already running (checked via session label), skip this tick
  │
  ▼
Blueprint session starts
  │
  ├─ 1. Ensure labels exist (ready-for-build, decomposed)
  ├─ 2. Create feature branch (if needed)
  ├─ 3. Spawn decomposition sub-agent (timeout: 10 min)
  │     │
  │     ├─ Post decomposition plan comment on plan issue
  │     ├─ Pass 1: Create task issues via gh issue create (depends_on: pending)
  │     ├─ Pass 2: Backfill depends_on with real issue numbers via gh issue edit
  │     └─ Return JSON result in final message
  │
  ├─ 4. Scope review: classify tasks, split/merge as needed
  ├─ 5. Update plan issue with task list comment (post-scope-review)
  ├─ 6. Label spec PR 'decomposed'
  └─ 7. Notify George via Telegram
```

### State Model

Labels drive state transitions:

```
Spec PR merged with plan:draft
  │
  ▼ (find-work.py detects: plan:draft present, decomposed absent)
Blueprint decomposes
  │
  ▼
Spec PR labeled 'decomposed' (alongside plan:draft)
Task issues labeled 'ready-for-build'
  │
  ▼ (BUILD's find-work.py picks up ready-for-build issues)
```

### Metadata Block Schema

```yaml
# Hidden HTML comment at top of issue body
attempts_build: 0      # Incremented by BUILD on each attempt
attempts_test: 0       # Incremented by TEST on each attempt
attempts_review: 0     # Incremented by REVIEW on each attempt
max_attempts: 3        # Retry limit per stage
feature_branch: string # Target branch for task PRs
spec: string           # Path to spec file in repo
plan: string           # Plan issue reference (e.g., #42)
pr: string             # PR number once created by BUILD
depends_on: string     # Comma-separated issue refs (e.g., #43,#44) or "pending" during Pass 1
decomp_id: string      # SHA256 hash for duplicate detection (spec_path:req_id:task_index)
```

### Integration Points

- **Cron wrapper → find-work.py:** Shell script runs Python, checks exit code
- **Cron wrapper → Blueprint session:** `openclaw session run` with task context from find-work.py stdout
- **Blueprint → Sub-agent:** Spec content + codebase context passed as task prompt
- **Sub-agent → GitHub:** Direct `gh issue create` and `gh issue edit` calls
- **Sub-agent → Blueprint:** JSON block in final message with `issues_created` and `dependency_map`
- **Blueprint → GitHub:** Label updates, comments, branch creation via `gh` CLI
- **Blueprint → Telegram:** Notification via OpenClaw messaging

### Decomposition Sub-Agent Contract

**Input (provided by Blueprint):**
- Full spec content (markdown)
- `AGENTS.md` content from target repo
- `.agents/commands.yml` content
- Directory listing of target repo (top 2 levels)
- Target repo name (e.g., `AIpexForge/snaphappy`)
- Feature branch name (e.g., `feat/add-webhook-handler`)
- Plan issue number (e.g., `42`)

**Output (JSON block in final message):**
```json
{
  "issues_created": [43, 44, 45, 46, 47],
  "dependency_map": {
    "44": [43],
    "45": [43, 44],
    "46": [45],
    "47": [45, 46]
  }
}
```

**Error output:**
```json
{
  "error": "gh issue create failed: HTTP 403",
  "issues_created": [43, 44],
  "issues_failed": ["Implement validation layer", "Add error handling"]
}
```

---

## Implementation Roadmap

1. **Phase 1: find-work.py** — Work discovery script
2. **Phase 2: Decomposition prompt** — `agents/plan/decompose/AGENTS.md`
3. **Phase 3: Blueprint integration** — Cron wrapper, sub-agent spawning, result handling
4. **Phase 4: Idempotency** — Hash-based duplicate detection, partial state recovery
5. **Phase 5: Notification** — Telegram summary

---

## Dependency Chain

```
REQ-001 (find-work.py)
  │
  ▼
REQ-004 (feature branch) ← depends on REQ-001
  │
  ▼
REQ-006 (decomposition prompt) ← depends on REQ-003
  │
  ▼
REQ-002 (sub-agent spawning) ← depends on REQ-001, REQ-004, REQ-008
  │
  ▼
REQ-003 (task issue structure) ← depends on REQ-004, REQ-006
  │
  ▼
REQ-005 (plan issue update + labels) ← depends on REQ-002, REQ-003
  │
  ▼
REQ-007 (scope review) ← depends on REQ-002, REQ-003, REQ-005
  │
  ▼
REQ-008 (idempotency) ← depends on REQ-001, REQ-003, REQ-005
  │
  ▼
REQ-009 (dependency ordering) ← depends on REQ-003, REQ-006
  │
  ▼
REQ-010 (notification) ← depends on REQ-005
```

---

## Out of Scope

- **Cross-spec dependencies** — v1 handles dependencies within a single spec only
- **Parallel decomposition** — v1 processes one spec per cron run
- **Task re-decomposition** — If a spec is updated after decomposition, manual intervention required
- **Taskmaster integration** — Decomposition uses native LLM reasoning
- **Subtask creation** — Tasks are flat (no subtasks within a task issue)
- **BUILD agent execution** — Separate spec
- **Cron job configuration** — Operational setup, not specced here
- **Label taxonomy creation** — ONBOARD creates the 6 core labels. Phase B only creates `decomposed` and verifies `ready-for-build` exists.

---

## Open Questions & Risks

| # | Question/Risk | Owner | Status |
|---|---------------|-------|--------|
| 1 | ~~Remote detection from local path~~ — Resolved: `git -C <path> remote get-url origin`, skip on failure | Blueprint | Resolved |
| 2 | ~~`decomposed` label replaces or joins `plan:draft`~~ — Resolved: added alongside, `plan:draft` never removed | George | Resolved |
| 3 | What happens if target repo's default branch has diverged significantly since spec was written? | Blueprint | Open |
| 4 | ~~Sub-agent timeout~~ — Resolved: 10 minutes | Blueprint | Resolved |
| 5 | Decomposition prompt quality may require 2-3 iterations against real specs before task sizing is reliable. First decomposition should be treated as a calibration run. | Blueprint | Risk (accepted) |

---

## Appendix: Task Breakdown Hints

| Task | Size | Agent Time Est. | Dependencies |
|------|------|-----------------|-------------|
| find-work.py (repo discovery + GitHub query + JSON output) | M (2h) | 30m | None |
| Decomposition sub-agent prompt (agents/plan/decompose/AGENTS.md) | L (3h) | 45m | None |
| Blueprint cron wrapper (shell script bridging find-work.py → session) | S (1h) | 20m | find-work.py |
| Blueprint cron integration (read context, assemble, spawn sub-agent) | M (2h) | 30m | Cron wrapper, decomposition prompt |
| Task issue body template + metadata block with hash generation | M (1.5h) | 30m | Decomposition prompt |
| Feature branch creation logic | S (30m) | 15m | None |
| Two-pass dependency backfill (create with pending, edit with real numbers) | M (1.5h) | 30m | Task issue template |
| Plan issue summary comment + label management | S (1h) | 20m | Sub-agent result handling |
| Hash-based duplicate detection + partial state recovery | M (2h) | 30m | Task issue template |
| Decomposition plan comment persistence | S (30m) | 10m | Decomposition prompt |
| Telegram notification template | S (30m) | 10m | Plan issue update |
| Post-decomposition scope review (classify, split/merge, dependency repair) | L (3h) | 45m | Sub-agent result handling |
| `decomposed` + `ready-for-build` label bootstrap | S (15m) | 5m | None |
