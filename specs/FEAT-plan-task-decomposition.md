# PRD: PLAN Phase B — Task Decomposition

**Author:** Blueprint (AI) + George Sapp
**Date:** 2026-03-01
**Status:** Draft
**Version:** 1.2
**Target Repo:** AIpexForge/agent-swarm
**Taskmaster Optimized:** No — uses native LLM decomposition

---

## Changes from v1.1 → v1.2

- **Dropped `decomp_id` everywhere** — Removed from metadata block schema, REQ-003 template, REQ-006 prompt spec, REQ-008 duplicate detection, REQ-002 testStrategy, and Technical Considerations. Duplicate detection now uses plan-comment + title matching.
- **Fixed circular dependencies** — REQ-003 depends on REQ-004 only (removed REQ-006). REQ-006 depends on REQ-003. REQ-008 depends on REQ-001 and REQ-003 only (removed REQ-005). REQ-002 depends on REQ-001, REQ-004 only (removed REQ-008). Dependency chain is now a strict DAG with no cycles.
- **REQ-007 moved to P0 and redesigned** — Scope review now occurs BEFORE issue creation. After sub-agent posts the decomposition plan comment, Blueprint reviews and approves the plan (classifying tasks, splitting/merging in the plan). Sub-agent then creates issues from the approved plan. No post-creation issue mutation.
- **Added `size` field to metadata block** — `size: small|medium|large` added to REQ-003 template and Technical Considerations schema.
- **Backfill recovery added** — REQ-008 now specifies: on retry, check existing task issues for `depends_on: pending`. If found, run Pass 2 (backfill) only; don't re-create issues.
- **REQ-005 split** — REQ-005a: Label Bootstrap (P0, before decomposition). REQ-005b: Plan Issue Update (P0, after issue creation). Both remain P0.
- **Minor fixes** — REQ-004 testStrategy uses `git merge-base --is-ancestor`. REQ-009 testStrategy softens `dep < num` to a warning. REQ-003/REQ-006 frame 200-line heuristic as a guideline (not hard limit). REQ-001 adds rate limit detection (403 → log warning).
- **Counts updated** — 9 P0 requirements, 2 P1 requirements. REQ-007 moved from P1 to P0; REQ-005 split into REQ-005a + REQ-005b.

---

## Changes from v1.0 → v1.1

- **REQ-007 (Scope Review) reinstated for v1** — Simplified from full LLM estimation to single-pass classification: small (<15m), medium (15-30m), large (>30m). Split tasks >60m, merge tightly-coupled <10m tasks. Rules-based with LLM judgment on sizing — not a separate estimation pipeline.
- **REQ-008 simplified** — Removed 50% threshold heuristic. Always complete remaining tasks on partial failure (never delete). Task plan persisted to plan issue comment before issue creation begins.
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

PLAN Phase B is Blueprint's cron-driven capability that detects merged spec PRs, decomposes them into task issues sized for ~30 minute BUILD agent execution, and creates a feature branch linking all tasks. It bridges the gap between "spec approved" and "BUILD has work to do." Without it, the pipeline stalls after planning — specs exist but no actionable tasks flow to BUILD. This spec covers 9 P0 requirements and 2 P1 requirements.

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
- **Dependencies:** REQ-001, REQ-002, REQ-003, REQ-004, REQ-005a, REQ-005b, REQ-006, REQ-007

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
  11. If a GitHub API call returns HTTP 403, logs a warning ("GitHub rate limit or auth error — skipping repo") and skips that repo without failing the entire run.
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

    # Rate limit case: mock 403 response
    # Assert: warning logged, repo skipped, other repos still processed
    ```
  - Edge cases to test: Multiple merged spec PRs (only oldest processed), PR with both `plan:draft` and `decomposed` (skipped), repo without `.agents/commands.yml` (skipped), repo with renamed remote (warning logged, skipped), GitHub API returns 403 (warning logged, repo skipped)
  - Performance threshold: Script completes in under 5 seconds for up to 10 repos
- **Task Breakdown:** Repo discovery + remote resolution (30m), GitHub query logic (30m), JSON output formatting (30m), exit code logic (15m), 403 handling (15m)
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
    # Verify based on default branch (no divergence)
    DEFAULT=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
    git fetch origin
    git merge-base --is-ancestor origin/$DEFAULT origin/feat/<slug>
    # exit 0 = feat branch contains all of default (not diverged)
    # Verify task metadata
    gh issue list --repo <target> --label ready-for-build --json body \
      | python3 -c "import json,sys; issues=json.load(sys.stdin); assert all('feature_branch: feat/<slug>' in i['body'] for i in issues)"
    ```
  - Edge cases to test: Branch already exists (reuse), default branch is not `main`, escalation path (comment on spec PR)
- **Task Breakdown:** Branch creation logic (15m), default branch detection (15m), idempotency check (15m), escalation cleanup (15m)
- **Dependencies:** REQ-001

#### REQ-005a: Label Bootstrap
- **Description:** Before decomposition begins, Blueprint ensures that the required labels exist on the target repo. This prevents `gh issue create --label` calls from failing silently.
- **Acceptance Criteria:**
  1. Blueprint checks for the existence of `ready-for-build` and `decomposed` labels on the target repo before any decomposition work begins.
  2. If either label is missing, Blueprint creates it via `gh label create --force` (idempotent).
  3. This step runs as the first action in every decomposition run, before feature branch creation or sub-agent spawning.
  4. `plan:draft` label is expected to already exist (created by ONBOARD); Blueprint does not create it.
- **testStrategy:**
  - Acceptance criteria: Labels exist before decomposition proceeds; creation is idempotent
  - Verification approach:
    ```bash
    # Delete labels if they exist, then run decomposition
    gh label delete ready-for-build --repo <target> --yes 2>/dev/null || true
    gh label delete decomposed --repo <target> --yes 2>/dev/null || true
    # Trigger decomposition
    # Assert: labels now exist
    gh label list --repo <target> --json name -q '.[].name' | grep -q 'ready-for-build'
    gh label list --repo <target> --json name -q '.[].name' | grep -q 'decomposed'
    ```
  - Edge cases to test: Labels already exist (no-op), only one label missing (creates missing one only)
- **Task Breakdown:** Label existence check (10m), creation logic (10m)
- **Dependencies:** REQ-001

#### REQ-003: Task Issue Structure
- **Description:** Each task issue created by the decomposition sub-agent follows a consistent structure with metadata block, description, acceptance criteria, testStrategy, and context links. The 200-line change estimate is a sizing guideline — prompt/config tasks may legitimately exceed it.
- **Acceptance Criteria:**
  1. Issue title format: `[TASK] <concise task description>`
  2. Issue body contains metadata block as hidden HTML comment:
     ```markdown
     <!-- agent-meta
     attempts_build: 0
     attempts_test: 0
     attempts_review: 0
     max_attempts: 3
     size: small|medium|large
     feature_branch: feat/<plan-slug>
     spec: specs/FEAT-<name>.md
     plan: #<plan-issue-number>
     pr:
     depends_on: #<issue>,#<issue>
     -->
     ```
  3. `size` is set from the scope review classification (REQ-007) before issue creation. Tasks reaching issue creation must have size ≤ medium.
  4. Issue body contains after metadata block:
     - **Description:** What this task accomplishes (2-5 sentences)
     - **Acceptance Criteria:** Numbered list pulled/derived from the spec's REQ acceptance criteria
     - **Test Strategy:** How BUILD's output will be validated, pulled from the spec's testStrategy
     - **Spec Reference:** Link to the spec file in the repo (e.g., `[spec](../../specs/FEAT-<name>.md)`)
     - **Files Likely Affected:** List of files/directories the task will probably touch (inferred from codebase scan)
     - **Dependencies:** Human-readable list of prerequisite task issues with links
  5. Issue is labeled `ready-for-build`
  6. Each task targets a single focused concern completable in ~30 min agent execution time. A task typically involves ~200 lines changed — treat this as a guideline, not a hard ceiling. Prompt/config tasks may exceed it legitimately.
- **testStrategy:**
  - Acceptance criteria: Every created issue matches the structure above, metadata is parseable, all links resolve
  - Verification approach:
    ```bash
    # For each task issue:
    body=$(gh issue view $NUM --repo <target> --json body -q .body)

    # Validate metadata block fields
    echo "$body" | python3 -c "
    import sys, re
    body = sys.stdin.read()
    meta = re.search(r'<!-- agent-meta\n(.*?)\n-->', body, re.DOTALL)
    assert meta, 'No metadata block'
    fields = dict(line.split(': ', 1) for line in meta.group(1).strip().split('\n') if ': ' in line)
    for key in ['attempts_build','attempts_test','attempts_review','max_attempts','size','feature_branch','spec','plan']:
        assert key in fields, f'Missing field: {key}'
    assert fields['size'] in ('small','medium','large'), f'Invalid size: {fields[\"size\"]}'
    assert fields['size'] != 'large', 'Large task should not reach issue creation without being split'
    "

    # Validate body sections
    for section in 'Description' 'Acceptance Criteria' 'Test Strategy' 'Spec Reference' 'Files Likely Affected'; do
      echo "$body" | grep -q "## $section\|### $section\|**$section"
    done

    # Validate label
    gh issue view $NUM --repo <target> --json labels -q '.labels[].name' | grep -q 'ready-for-build'
    ```
  - Edge cases to test: Task with no dependencies (depends_on is empty), task referencing a file that doesn't exist yet (acceptable), spec with only 1 requirement (produces 1-2 tasks), prompt/config task exceeding 200 lines (acceptable with size ≤ medium)
- **Task Breakdown:** Issue body template (1h), metadata block generation with size field (30m), acceptance criteria extraction (1h), file inference (30m)
- **Dependencies:** REQ-004

#### REQ-006: Decomposition Prompt
- **Description:** The prompt that drives the decomposition sub-agent. It instructs the LLM to read a spec, understand the codebase, post a plan comment for scope review approval, then (after approval) create appropriately-scoped task issues directly via `gh issue create`. The 200-line change estimate is a sizing guideline — the prompt should acknowledge that prompt/config tasks may legitimately exceed it.
- **Acceptance Criteria:**
  1. Prompt file exists at `agents/plan/decompose/AGENTS.md`
  2. Prompt includes:
     - Role: "You are a task decomposition agent"
     - Input specification: spec content, codebase context, target repo, feature branch, plan issue number
     - Task sizing guidance: each task should be completable by BUILD in ~30 minutes (single focused concern, ~200 lines changed as a guideline — prompt/config tasks may exceed this)
     - Issue structure template (matching REQ-003 exactly, including the `size` field)
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
  5. Prompt instructs: before creating any issues, post a "decomposition plan" comment on the plan issue listing intended tasks (titles, estimated size, order). Then WAIT for Blueprint approval before calling `gh issue create` for any task.
  6. Prompt instructs: after receiving Blueprint approval with the final task list, proceed to create issues from the approved list (which may differ from the original plan due to scope review adjustments).
- **testStrategy:**
  - Acceptance criteria: Prompt file exists, contains all required sections, sub-agent can execute from it
  - Verification approach: Load prompt in a test session with the ONBOARD spec (`specs/FEAT-onboard-capability.md`) against the agent-swarm repo. Verify: (a) decomposition plan comment appears on plan issue before task creation, (b) sub-agent pauses and waits for Blueprint approval, (c) task issues are created with valid structure per REQ-003 only after approval, (d) `depends_on` fields are backfilled with real issue numbers after all issues exist, (e) final message contains valid JSON with `issues_created` and `dependency_map`
  - Edge cases to test: Spec with only P1 requirements (no P0), spec with complex dependency chains between requirements
- **Task Breakdown:** Prompt writing (2h), testing and iteration (1h)
- **Dependencies:** REQ-003

#### REQ-007: Pre-Creation Scope Review
- **Description:** After the decomposition sub-agent posts a plan comment listing intended tasks (but BEFORE creating any issues), Blueprint reviews and approves the plan. Blueprint classifies each proposed task as small (<15 min), medium (15-30 min), or large (>30 min) by estimated BUILD agent execution time, adjusts the plan as needed (splitting large tasks, merging tightly-coupled small ones in the plan), then signals approval. The sub-agent creates issues only from the approved plan. No post-creation issue mutation occurs.
- **Acceptance Criteria:**
  1. After sub-agent posts the decomposition plan comment, Blueprint reviews it before any `gh issue create` calls are made.
  2. Blueprint classifies each proposed task as small / medium / large based on estimated agent minutes.
  3. Proposed tasks estimated at >60 min are split into smaller tasks in the plan (adjusting titles and scoping acceptance criteria). New tasks are listed in the updated plan.
  4. Tightly-coupled tasks (<10 min each, sequential dependency, touching the same files) may be merged in the plan before issue creation.
  5. Blueprint posts an updated plan comment (or edits the existing one) showing size classifications and any splits/merges made.
  6. Blueprint signals approval to the sub-agent (via a follow-up message) with the final approved task list.
  7. Sub-agent creates issues only from the approved task list. No task classified as large (>60 min) is created as a single issue — it must be split in the plan first.
  8. If no tasks need adjustment, scope review is a no-op — Blueprint approves the plan as-is.
- **testStrategy:**
  - Acceptance criteria: Plan comment contains size classifications before issue creation; no task classified large makes it to issue creation without being split; dependency chain after creation is valid
  - Verification approach:
    ```bash
    # Verify plan comment was posted with size classifications before any issues existed
    comment=$(gh issue view <plan> --repo <target> --json comments -q '.comments[] | select(.body | contains("size:")) | .body' | head -1)
    echo "$comment" | grep -qE 'small|medium|large'

    # Verify no created issue has size: large
    gh issue list --repo <target> --label ready-for-build --json body \
      | python3 -c "
    import json, sys, re
    issues = json.loads(sys.stdin.read())
    for i in issues:
        meta = re.search(r'size: (\w+)', i['body'])
        assert meta, 'Missing size field in issue body'
        assert meta.group(1) != 'large', 'Large task reached issue creation without being split'
    "

    # Verify dependency DAG is acyclic after creation
    # (reuse REQ-009 DAG validation)
    ```
  - Edge cases to test: All tasks medium (no-op approval), one large task requiring split (plan updated, two issues created instead of one), two small tasks merged (one issue created), plan adjustment changes dependency order
- **Task Breakdown:** Scope classification logic (30m), plan comment review + update (30m), approval signaling to sub-agent (15m), split/merge in plan (45m)
- **Dependencies:** REQ-006

#### REQ-002: Decomposition Sub-Agent Spawning
- **Description:** Blueprint spawns a focused sub-agent for each spec that needs decomposition. The sub-agent posts a plan comment, waits for Blueprint's scope review approval (REQ-007), then creates task issues directly via `gh issue create` from the approved plan.
- **Acceptance Criteria:**
  1. Blueprint reads the merged spec file and relevant codebase context (AGENTS.md, commands.yml, directory structure)
  2. Blueprint spawns a sub-agent with: spec content, codebase context, decomposition prompt, target repo, feature branch name, plan issue number
  3. Issue creation does not begin until Blueprint approves the decomposition plan (REQ-007). The sub-agent waits for the approval signal before calling `gh issue create`.
  4. The sub-agent creates task issues directly using `gh issue create` (no intermediate file handoff)
  5. The sub-agent returns a JSON block in its final message with created issue numbers and dependency map
  6. If the sub-agent fails or returns malformed output, Blueprint retries once with the same inputs (running duplicate detection per REQ-008 first to avoid re-creating existing issues)
  7. If retry fails, Blueprint labels the spec PR `escalated` and notifies George via Telegram
  8. Sub-agent timeout: 10 minutes. If exceeded, Blueprint treats it as a failure and follows the retry path.
- **testStrategy:**
  - Acceptance criteria: Sub-agent receives correct context, waits for scope review approval, creates valid issues, returns issue number list
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
      echo "$body" | grep -q 'size:'
    done
    ```
  - Edge cases to test: Sub-agent fails mid-creation (partial issues exist), sub-agent returns empty list, sub-agent times out at 10 min, sub-agent creates issues before scope review approval (must not happen — prompt enforces the wait)
- **Task Breakdown:** Context assembly (1h), sub-agent prompt (2h), spawn + result handling (1h), retry logic with duplicate detection (30m)
- **Dependencies:** REQ-001, REQ-004

#### REQ-008: Idempotency and Partial State Recovery
- **Description:** The decomposition process must be idempotent — running it twice on the same spec must not create duplicate issues. If decomposition fails partway through, the next run must detect and handle the partial state.
- **Acceptance Criteria:**
  1. `find-work.py` skips specs with the `decomposed` label (primary idempotency gate)
  2. Before creating issues, the sub-agent posts a decomposition plan comment on the plan issue (see REQ-006 AC5). This persists the intended task list for recovery.
  3. If decomposition fails after creating some issues but before adding the `decomposed` label:
     - Blueprint reads the plan issue's decomposition plan comment to determine the full intended task list
     - Blueprint queries existing task issues for this spec by searching for issues whose titles match those in the plan comment
     - Blueprint identifies which tasks already exist (title match) and creates only missing ones
     - Blueprint never deletes existing issues
     - Once all tasks exist, adds `decomposed` label
  4. On retry: before re-invoking the sub-agent, check existing task issues for `depends_on: pending`. If found, all issues already exist and only the backfill pass is missing — run Pass 2 (backfill) only, do not re-create any issues.
  5. Duplicate detection uses plan-comment + title matching: before creating an issue, check if an issue with the same title already exists:
     ```bash
     gh issue list --repo <target> --search "in:title [TASK] <title>" --json number
     ```
  6. All state transitions (label additions, issue creation) are logged as comments for debugging
- **testStrategy:**
  - Acceptance criteria: No duplicate issues created on re-run, partial state is detected and completed, backfill-only path triggered correctly
  - Verification approach:
    ```bash
    # Test 1: Full idempotency — run twice
    # First run: decomposition completes, decomposed label added
    # Second run: find-work.py exits 1 (no work)
    python3 agents/plan/scripts/find-work.py; echo "exit: $?"  # should be 1

    # Test 2: Partial failure recovery
    # Simulate: create 2 of 5 tasks manually (with correct titles matching plan comment), don't add decomposed label
    # Run decomposition: should create only the 3 missing tasks
    gh issue list --repo <target> --label ready-for-build --json number | python3 -c "import json,sys; assert len(json.load(sys.stdin)) == 5"

    # Test 3: Backfill-only path
    # Simulate: all 5 issues exist but all have depends_on: pending
    # Run decomposition retry: should run Pass 2 only (gh issue edit), not create new issues
    # Verify total count unchanged
    gh issue list --repo <target> --label ready-for-build --json number | python3 -c "import json,sys; assert len(json.load(sys.stdin)) == 5"
    # Verify no pending depends_on values remain
    gh issue list --repo <target> --label ready-for-build --json body \
      | python3 -c "import json,sys; issues=json.load(sys.stdin); assert not any('depends_on: pending' in i['body'] for i in issues)"
    ```
  - Edge cases to test: Decomposition fails after creating 0 issues (clean re-run from plan comment), fails after all issues but before labeling (just add label), all issues exist with `depends_on: pending` (backfill only), concurrent repo activity creating issues between tasks
- **Task Breakdown:** Plan-comment + title-based duplicate detection (30m), decomposition plan persistence (30m), partial state detection + completion (1h), backfill-only retry path (30m), logging (15m)
- **Dependencies:** REQ-001, REQ-003

#### REQ-005b: Plan Issue Update and Label Management
- **Description:** After scope review and issue creation are complete, Blueprint adds a summary comment to the plan issue listing all created task issues and labels the spec PR `decomposed`.
- **Acceptance Criteria:**
  1. Comment on the plan issue contains:
     - Number of tasks created
     - Numbered list of task issues with titles, sizes, and links
     - Dependency graph (simple text: "Task #N depends on #M")
     - Feature branch name
  2. Comment uses the standard agent comment format:
     ```markdown
     <details>
     <summary>📐 PLAN — Decomposed spec into N tasks (Xm Ys)</summary>

     **Feature branch:** `feat/<slug>`
     **Tasks created:**
     1. #43 — Implement user auth endpoint (small)
     2. #44 — Add validation middleware (medium, depends on #43)
     ...

     </details>
     ```
  3. The spec PR receives the `decomposed` label (added alongside existing `plan:draft`)
  4. `plan:draft` label is NOT removed — it must remain present after `decomposed` is added.
- **testStrategy:**
  - Acceptance criteria: Plan issue has summary comment with all required fields, spec PR has `decomposed` label, `plan:draft` still present
  - Verification approach:
    ```bash
    # Verify comment on plan issue
    comments=$(gh issue view <plan> --repo <target> --json comments -q '.comments[-1].body')
    echo "$comments" | grep -q 'PLAN — Decomposed spec into'
    echo "$comments" | grep -q 'feat/'
    echo "$comments" | grep -qP '#\d+'        # at least one issue link
    echo "$comments" | grep -qE 'small|medium' # size classifications present

    # Verify decomposed label on spec PR
    gh pr view <spec-pr> --repo <target> --json labels -q '.labels[].name' | grep -q 'decomposed'

    # Verify plan:draft still present
    gh pr view <spec-pr> --repo <target> --json labels -q '.labels[].name' | grep -q 'plan:draft'
    ```
  - Edge cases to test: Plan issue already has comments from Phase A (new comment appends), labels don't exist (auto-created by REQ-005a — should not reach this step without them)
- **Task Breakdown:** Comment template with size fields (30m), PR label update (15m), dependency summary text (15m)
- **Dependencies:** REQ-002, REQ-003, REQ-007

---

### P1 — Should Have

#### REQ-009: Dependency-Aware Task Ordering
- **Description:** Tasks are created in dependency order so that prerequisite tasks have lower issue numbers. The `depends_on` field references real GitHub issue numbers, backfilled after all issues exist.
- **Acceptance Criteria:**
  1. Pass 1: Sub-agent creates all tasks in logical dependency order (foundations first), with `depends_on: pending` in metadata
  2. Pass 2: After all issues are created, sub-agent updates each issue's metadata block via `gh issue edit`, replacing `pending` with actual issue number references
  3. No circular dependencies exist in the created task set
  4. BUILD can use `depends_on` to skip tasks whose prerequisites aren't labeled `complete`
- **testStrategy:**
  - Acceptance criteria: Task issues are numbered in dependency order, `depends_on` references are valid after backfill, DAG is acyclic
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
    # Hard assertion: verify DAG (no cycles)
    visited, stack = set(), set()
    def has_cycle(n):
        if n in stack: return True
        if n in visited: return False
        stack.add(n); visited.add(n)
        for dep in graph.get(n, []):
            if has_cycle(dep): return True
        stack.discard(n); return False
    assert not any(has_cycle(n) for n in graph), 'Circular dependency detected'
    # Soft check (warning only): deps should ideally have lower issue numbers
    for num, deps in graph.items():
        for dep in deps:
            if dep >= num:
                print(f'WARNING: Issue #{num} depends on #{dep} (higher number — non-sequential creation order, not an error)')
    " <<< "\$issues"
    ```
  - Edge cases to test: Spec where all tasks are independent (no dependencies), spec with deep chain (A → B → C → D), concurrent issue creation on repo causing non-sequential numbers (valid — warning only, not a failure)
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
- **Dependencies:** REQ-005b

---

## Non-Functional Requirements

- **Performance:** `find-work.py` completes in under 5 seconds for up to 10 repos. Full decomposition (sub-agent spawn, scope review, issue creation, backfill) completes in under 10 minutes per spec.
- **Cost efficiency:** Zero token spend when no work exists. `find-work.py` uses only `gh` CLI calls (GitHub API), no LLM.
- **Reliability:** Idempotent — safe to run repeatedly. Partial failures are detected and completed on next run. No duplicate issues under any circumstances.
- **Security:** Sub-agent has access only to `gh` CLI and file read tools. Does not execute code from the target repo.
- **Prerequisites:** Target repo must be onboarded (`.agents/commands.yml` exists). `gh` CLI authenticated with push access. Labels auto-created by REQ-005a if missing. `plan:draft` label exists (created by ONBOARD).

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
  ├─ 1. Ensure labels exist (REQ-005a: ready-for-build, decomposed)
  ├─ 2. Create feature branch if needed (REQ-004)
  ├─ 3. Spawn decomposition sub-agent (REQ-002, timeout: 10 min)
  │     │
  │     └─ Sub-agent posts decomposition plan comment on plan issue
  │          (lists proposed tasks with titles + estimated sizes)
  │          Sub-agent WAITS for Blueprint approval
  │
  ├─ 4. Blueprint scope review (REQ-007):
  │     ├─ Classify each task: small / medium / large
  │     ├─ Split large tasks (>60m) in the plan
  │     ├─ Merge tightly-coupled small tasks in the plan
  │     ├─ Update plan comment with classifications
  │     └─ Signal approval to sub-agent with final approved task list
  │
  ├─ 5. Sub-agent proceeds (REQ-002, REQ-006):
  │     ├─ Pass 1: Create task issues via gh issue create (depends_on: pending)
  │     ├─ Pass 2: Backfill depends_on with real issue numbers via gh issue edit
  │     └─ Return JSON result in final message
  │
  ├─ 6. Post summary comment on plan issue (REQ-005b)
  ├─ 7. Label spec PR 'decomposed' (REQ-005b)
  └─ 8. Notify George via Telegram (REQ-010)
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
size: small            # small | medium | large — set by scope review (REQ-007); always ≤ medium at issue creation
feature_branch: string # Target branch for task PRs
spec: string           # Path to spec file in repo
plan: string           # Plan issue reference (e.g., #42)
pr: string             # PR number once created by BUILD
depends_on: string     # Comma-separated issue refs (e.g., #43,#44) or "pending" during Pass 1
```

### Integration Points

- **Cron wrapper → find-work.py:** Shell script runs Python, checks exit code
- **Cron wrapper → Blueprint session:** `openclaw session run` with task context from find-work.py stdout
- **Blueprint → Sub-agent:** Spec content + codebase context passed as task prompt
- **Sub-agent → Blueprint:** Plan comment posted to plan issue; sub-agent awaits Blueprint scope review approval
- **Blueprint → Sub-agent:** Approval signal with final approved task list after scope review
- **Sub-agent → GitHub:** Direct `gh issue create` and `gh issue edit` calls (only after approval)
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

**Phase 1 output (plan comment posted to plan issue):**
Proposed task list with titles, estimated sizes (small/medium/large), and intended dependency order. Sub-agent waits here for Blueprint approval.

**Phase 2 input (Blueprint approval message):**
Final approved task list (may differ from Phase 1 proposal due to scope review adjustments).

**Final output (JSON block in final message):**
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
3. **Phase 3: Blueprint integration** — Cron wrapper, sub-agent spawning, scope review (pre-creation), result handling
4. **Phase 4: Idempotency** — Title-based duplicate detection, partial state recovery, backfill-only retry
5. **Phase 5: Notification** — Telegram summary

---

## Dependency Chain

```
REQ-001 (find-work.py) — no deps
  │
  ├─▶ REQ-004 (feature branch) — REQ-001
  │     │
  │     └─▶ REQ-003 (task issue structure) — REQ-004
  │               │
  │               ├─▶ REQ-006 (decomposition prompt) — REQ-003
  │               │     │
  │               │     └─▶ REQ-007 (pre-creation scope review) — REQ-006
  │               │
  │               └─▶ REQ-008 (idempotency) — REQ-001, REQ-003
  │
  └─▶ REQ-005a (label bootstrap) — REQ-001

REQ-001 + REQ-004
  └─▶ REQ-002 (sub-agent spawning) — REQ-001, REQ-004

REQ-002 + REQ-003 + REQ-007
  └─▶ REQ-005b (plan issue update) — REQ-002, REQ-003, REQ-007

REQ-003 + REQ-006
  └─▶ REQ-009 (P1: dependency ordering) — REQ-003, REQ-006

REQ-005b
  └─▶ REQ-010 (P1: notification) — REQ-005b
```

**Verified DAG — complete dependency list (no cycles):**

| Requirement | Depends On |
|-------------|------------|
| REQ-001 | — |
| REQ-004 | REQ-001 |
| REQ-005a | REQ-001 |
| REQ-003 | REQ-004 |
| REQ-006 | REQ-003 |
| REQ-007 | REQ-006 |
| REQ-002 | REQ-001, REQ-004 |
| REQ-008 | REQ-001, REQ-003 |
| REQ-005b | REQ-002, REQ-003, REQ-007 |
| REQ-009 (P1) | REQ-003, REQ-006 |
| REQ-010 (P1) | REQ-005b |

---

## Out of Scope

- **Cross-spec dependencies** — v1 handles dependencies within a single spec only
- **Parallel decomposition** — v1 processes one spec per cron run
- **Task re-decomposition** — If a spec is updated after decomposition, manual intervention required
- **Taskmaster integration** — Decomposition uses native LLM reasoning
- **Subtask creation** — Tasks are flat (no subtasks within a task issue)
- **Post-creation issue mutation** — Scope review happens before issue creation; issues are never split/merged after creation
- **BUILD agent execution** — Separate spec
- **Cron job configuration** — Operational setup, not specced here
- **Label taxonomy creation** — ONBOARD creates the 6 core labels. Phase B only creates `decomposed` and verifies `ready-for-build` exists (REQ-005a).

---

## Open Questions & Risks

| # | Question/Risk | Owner | Status |
|---|---------------|-------|--------|
| 1 | ~~Remote detection from local path~~ — Resolved: `git -C <path> remote get-url origin`, skip on failure | Blueprint | Resolved |
| 2 | ~~`decomposed` label replaces or joins `plan:draft`~~ — Resolved: added alongside, `plan:draft` never removed | George | Resolved |
| 3 | What happens if target repo's default branch has diverged significantly since spec was written? | Blueprint | Open |
| 4 | ~~Sub-agent timeout~~ — Resolved: 10 minutes | Blueprint | Resolved |
| 5 | Decomposition prompt quality may require 2-3 iterations against real specs before task sizing is reliable. First decomposition should be treated as a calibration run. | Blueprint | Risk (accepted) |
| 6 | Sub-agent "wait for approval" interaction model depends on session communication mechanism — needs verification against openclaw session run capabilities. | Blueprint | Open |

---

## Appendix: Task Breakdown Hints

| Task | Size | Agent Time Est. | Dependencies |
|------|------|-----------------|-------------|
| find-work.py (repo discovery + GitHub query + JSON output + 403 handling) | M (2h) | 30m | None |
| Decomposition sub-agent prompt (agents/plan/decompose/AGENTS.md) | L (3h) | 45m | None |
| Blueprint cron wrapper (shell script bridging find-work.py → session) | S (1h) | 20m | find-work.py |
| Blueprint cron integration (read context, assemble, spawn sub-agent) | M (2h) | 30m | Cron wrapper, decomposition prompt |
| Task issue body template + metadata block with size field | M (1.5h) | 30m | Decomposition prompt |
| Feature branch creation logic | S (30m) | 15m | None |
| Label bootstrap — REQ-005a (ready-for-build, decomposed) | S (15m) | 5m | None |
| Pre-creation scope review (classify, update plan comment, approve) | M (1.5h) | 30m | Plan comment from sub-agent |
| Two-pass dependency backfill (create with pending, edit with real numbers) | M (1.5h) | 30m | Task issue template |
| Plan issue summary comment + decomposed label (REQ-005b) | S (1h) | 20m | Sub-agent result handling |
| Title-based duplicate detection + partial state recovery + backfill-only retry | M (2h) | 30m | Task issue template |
| Decomposition plan comment persistence (pre-creation) | S (30m) | 10m | Decomposition prompt |
| Telegram notification template | S (30m) | 10m | Plan issue update |
