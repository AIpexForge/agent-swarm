# Spec: Agent Workflow System

> **Plan:** Initial system design  
> **Status:** Draft — awaiting human approval  
> **Author:** Servo (AI) + George Sapp  
> **Date:** 2026-02-27

---

## Overview

OpenClaw-Workflow is a collection of independent, decoupled AI agents that perform discrete functions in a software development lifecycle. Unlike monolithic orchestrators, each agent has a single responsibility, clear input/output contracts, and communicates through GitHub's native primitives (issues, PRs, labels, comments).

**Design principles:**
- **Agent-agnostic** — Nothing in the target repo structure is tied to OpenClaw. Any agent framework can operate on these conventions.
- **GitHub as shared state** — Issues, PRs, and labels are the coordination layer. No external database.
- **Session isolation** — Each agent invocation starts a fresh session. No long-running processes.
- **Retry with escalation** — Agents retry up to 3 times per task, then escalate to a human.
- **Sequential in v1** — Tasks are processed one at a time. Architecture supports future parallelization.

---

## Agents

### ONBOARD

**Purpose:** Prepare a repository for agent operation.

**Trigger:** Manual invocation.

**Actions:**
1. Clone and analyze the target repo (detect stack, frameworks, tooling)
2. Generate `.agents/commands.yml` with build, test, dev, and deploy commands
3. Create or update `AGENTS.md` referencing the commands file
4. Create `specs/`, `specs/archive/`, and `plans/` directories
5. Move any existing spec-like documents into `specs/`
6. Set up the label taxonomy on the GitHub repo (see [Labels](#labels))
7. Open a PR for human review

**Output:** PR with repo scaffolding.

---

### PLAN

**Purpose:** Conduct planning interviews, produce plans, specs, and task issues.

**Trigger:** Cron at :00 (every 30 min). Also invoked interactively when user initiates planning.

**Phase A — Plan + Spec Creation (interactive):**
1. Interview the user (JTBD-style discovery)
2. Create a plan file in `plans/` (high-level what/why)
3. Create a spec file in `specs/` (detailed how — architecture, acceptance criteria, test requirements, happy path definition)
4. Create a plan issue on GitHub linking to both files
5. Open a PR containing the plan + spec files, labeled `plan:draft`
6. Notify user for review

**Phase B — Task Decomposition (cron-driven):**
1. Detect merged spec PRs that have no associated task issues
2. Break the spec into task issues (each task = one focused PR, max ~200 lines)
3. Create a feature branch: `feat/<plan-slug>`
4. Link all task issues to the plan issue
5. Add metadata block to each task issue (see [Issue Metadata](#issue-metadata))
6. Label all task issues `ready-for-build`

**Planning approach:** Uses proven external planning methodologies (deep-project, deep-plan from the [Deep Trilogy](https://pierce-lamb.medium.com/the-deep-trilogy-claude-code-plugins-for-writing-good-software-fast-33b76f2a022d)) rather than custom planning logic. Specific skill selection is an open item (see [Open Items](#open-items)).

**Human gate:** Human approves by merging the spec PR. No work begins until merge.

---

### BUILD

**Purpose:** Write code and submit PRs for task issues.

**Trigger:** Cron at :10 (every 30 min).

**Input:**
- GitHub issues labeled `ready-for-build` (new tasks)
- GitHub PRs labeled `ready-for-build` (PRs with feedback from TEST or REVIEW)

**Actions:**
1. Pick the highest-priority `ready-for-build` item (oldest first, or by issue priority)
2. Read the linked spec and plan for context
3. Read `.agents/commands.yml` for build/test commands
4. Clone the repo, checkout the feature branch
5. Write code, run tests locally (`test_cmd` from commands.yml)
6. Commit and push to a task branch off the feature branch
7. Open a PR targeting the feature branch
8. Update the task issue metadata block (status, attempt count, PR link)
9. Comment on the task issue with a summary of work done
10. Label the PR `ready-for-test`

**On feedback (PR labeled `ready-for-build`):**
1. Read review comments on the PR
2. Address feedback, push new commits
3. Re-label PR `ready-for-test`
4. Increment attempt counter

**Retry:** 3 attempts max. On failure, label `escalated`, @-mention human on the issue, notify via Telegram.

---

### TEST

**Purpose:** Validate PRs through automated and manual testing.

**Trigger:** Cron at :20 (every 30 min). Also triggered on production deployments (via GitHub Action).

**Input:**
- GitHub PRs labeled `ready-for-test`
- Issues labeled `ready-for-test` (production deployment validation)

**Actions — PR Testing:**
1. Read the linked spec for acceptance criteria and happy path definition
2. Review test coverage — identify missing tests for the change
3. Run automated test suite (`test_cmd` from commands.yml)
4. Start the application locally (`dev_cmd` from commands.yml)
5. Execute manual happy-path testing using Playwright MCP and Chrome DevTools MCP
6. If tests pass:
   - Label PR `ready-for-review`
   - Comment with test results summary
7. If tests fail:
   - Post screenshots of failures to the PR
   - Label PR `ready-for-build`
   - Comment with failure details
   - Increment attempt counter on task issue

**Actions — Feature Branch Testing (post all-tasks-merged):**
1. Detect feature branches where all task PRs are merged
2. Run full test suite on the feature branch
3. Execute full happy-path manual testing
4. If passing: comment on plan issue recommending deployment
5. If failing: create bug issues linked to the plan, labeled `ready-for-build`

**Actions — Production Deployment Validation:**
1. Triggered by GitHub Action on manual deploy
2. Run happy-path tests against production URL
3. Post results to the deployment issue/PR
4. If critical failures: create hotfix issues labeled `ready-for-build` with high priority

**Retry:** 3 attempts max per PR. Escalates on persistent failure.

---

### REVIEW

**Purpose:** Code review for quality, consistency, and spec compliance.

**Trigger:** Cron at :30 (every 30 min).

**Input:** GitHub PRs labeled `ready-for-review`.

**Actions:**
1. Read the linked spec and plan for context
2. Review the PR diff for:
   - Code quality and consistency
   - Spec compliance (does the code match acceptance criteria?)
   - Security concerns
   - Performance implications
   - Documentation completeness
3. If approved:
   - Approve the PR on GitHub
   - Label PR `ready-for-merge`
   - Comment with review summary
4. If changes needed:
   - Request changes on the PR
   - Label PR `ready-for-build`
   - Comment with specific, actionable feedback on the PR diff
   - Increment attempt counter on task issue

**Retry:** 3 review cycles max per PR. Escalates if BUILD can't satisfy review.

**Skill:** Uses proven code review methodology (see [Open Items](#open-items)).

---

### MERGE

**Purpose:** Merge approved PRs and manage feature branch lifecycle.

**Trigger:** Cron at :40 (every 30 min).

**Input:** GitHub PRs labeled `ready-for-merge`.

**Actions — Task PR Merge:**
1. Check for merge conflicts with the feature branch
2. If conflicts: rebase/resolve using context from recent merges (tracked in agent memory)
3. Merge the PR into the feature branch (squash merge)
4. Update task issue metadata (status: complete)
5. Comment on task issue with merge confirmation
6. Check if all tasks for the plan are now complete

**Actions — Feature Branch Finalization (after all tasks + TEST approval):**
1. Confirm TEST agent has approved the feature branch
2. Merge feature branch into main (merge commit, preserve history)
3. Move spec from `specs/` to `specs/archive/`
4. Close the plan issue
5. Comment on plan issue with final summary
6. Notify human that feature is ready for deployment

**Conflict resolution:** MERGE maintains memory of recent merges to inform conflict resolution decisions. If a conflict can't be auto-resolved, escalates to human.

**Retry:** 3 attempts at conflict resolution. Escalates if unresolvable.

---

## Workflow Lifecycle

```
User idea
  │
  ▼
PLAN (interactive) ──→ Plan issue + Spec PR + Plan file
  │
  ▼
Human reviews & merges spec PR
  │
  ▼
PLAN (cron) ──→ Task issues (ready-for-build) + Feature branch
  │
  ▼
BUILD ──→ Task PR (ready-for-test) ──→ TEST
  │                                       │
  │                    ┌──── pass ────────┘
  │                    │         │
  │                    ▼         ▼ fail
  │               REVIEW     BUILD (retry)
  │                 │
  │      ┌── pass ──┘── fail ──→ BUILD (retry)
  │      ▼
  │    MERGE ──→ PR merged to feature branch
  │      │
  │      ▼
  │   All tasks done?
  │      │
  │      yes ──→ TEST (feature branch) ──→ pass ──→ MERGE (to main)
  │                    │                              │
  │                    fail ──→ Bug issues            ▼
  │                              (ready-for-build)   Human deploys
  │
  ▼ (3 failures)
ESCALATE ──→ @human on issue + Telegram notification
```

---

## GitHub State Model

### Labels

| Label | Applied by | Meaning |
|-------|-----------|---------|
| `plan:draft` | PLAN | Plan+spec PR under review |
| `ready-for-build` | PLAN, TEST, REVIEW | Task/PR ready for BUILD agent |
| `ready-for-test` | BUILD | PR ready for TEST agent |
| `ready-for-review` | TEST | PR ready for REVIEW agent |
| `ready-for-merge` | REVIEW | PR ready for MERGE agent |
| `escalated` | Any agent | Human intervention required |

### Issue Metadata

Every task issue contains a metadata block at the top of the issue body:

```markdown
<!-- agent-meta
status: ready-for-build
attempts: 0
max_attempts: 3
feature_branch: feat/add-webhook-handler
spec: specs/FEAT-add-webhook-handler.md
plan: #42
pr: 
-->
```

Agents update this block directly on the issue body. Fields:
- `status` — Current state (`ready-for-build`, `in-progress`, `ready-for-test`, `ready-for-review`, `ready-for-merge`, `complete`, `escalated`)
- `attempts` — Number of build/test/review cycles
- `max_attempts` — Retry limit (default: 3)
- `feature_branch` — Target branch for PRs
- `spec` — Path to the spec file in the repo
- `plan` — Issue number of the parent plan
- `pr` — PR number once created

### Agent Comment Format

Agents log work using a standard collapsible format:

```markdown
<details>
<summary>🤖 BUILD — Implemented user authentication endpoint (2m 34s)</summary>

**Action:** Created PR #15 implementing `/api/auth/login` endpoint
**Files changed:** 3 (src/auth.js, src/routes.js, tests/auth.test.js)
**Tests:** 12 added, all passing
**Next:** Ready for TEST

</details>
```

---

## Target Repo Structure

Each repository onboarded into the workflow gets:

```
repo/
├── .agents/
│   └── commands.yml        # Build, test, dev, deploy commands
├── plans/
│   └── PLAN-feature-name.md    # Plan documents (persistent)
├── specs/
│   ├── FEAT-feature-name.md    # Active specs
│   └── archive/                # Completed specs
│       └── FEAT-old-feature.md
└── AGENTS.md                   # References .agents/commands.yml, repo conventions
```

### `.agents/commands.yml`

```yaml
# Agent-agnostic command definitions for CI/CD and local development
stack: nextjs  # or: rails, django, express, etc.

commands:
  install: npm install
  build: npm run build
  test: npm test
  dev: npm run dev
  lint: npm run lint

environment:
  port: 3000
  node_version: "20"
  env_file: .env.local  # Template or example env file

# Optional: paths agents should know about
paths:
  source: src/
  tests: tests/
  docs: docs/
```

---

## OpenClaw-Workflow Repo Structure

```
OpenClaw-Workflow/
├── agents/
│   ├── onboard/
│   │   ├── agent.yml           # OpenClaw agent config
│   │   ├── prompt.md           # System prompt
│   │   └── scripts/
│   │       └── onboard.sh      # Repo analysis + scaffolding
│   ├── plan/
│   │   ├── agent.yml
│   │   ├── prompt.md
│   │   └── scripts/
│   │       └── find-work.sh    # Detect merged specs without tasks
│   ├── build/
│   │   ├── agent.yml
│   │   ├── prompt.md
│   │   └── scripts/
│   │       └── find-work.sh    # Find ready-for-build issues/PRs
│   ├── test/
│   │   ├── agent.yml
│   │   ├── prompt.md
│   │   └── scripts/
│   │       └── find-work.sh    # Find ready-for-test PRs
│   ├── review/
│   │   ├── agent.yml
│   │   ├── prompt.md
│   │   └── scripts/
│   │       └── find-work.sh    # Find ready-for-review PRs
│   └── merge/
│       ├── agent.yml
│       ├── prompt.md
│       └── scripts/
│           └── find-work.sh    # Find ready-for-merge PRs
├── skills/                     # Proven skills injected into agent prompts
│   └── README.md
├── config/
│   ├── repos.yml               # AIpexForge repo allow-list
│   └── cron.yml                # Cron schedule definitions
├── specs/
│   └── FEAT-agent-workflow-system.md  # This file
├── plans/
│   └── PLAN-agent-workflow-system.md
├── AGENTS.md
└── README.md
```

---

## Cron Schedule

All times are every 30 minutes, staggered:

| Agent  | Schedule | Example runs |
|--------|----------|-------------|
| PLAN   | :00, :30 | 9:00, 9:30, 10:00 |
| BUILD  | :10, :40 | 9:10, 9:40, 10:10 |
| TEST   | :20, :50 | 9:20, 9:50, 10:20 |
| REVIEW | :30, :00 | 9:30, 10:00, 10:30 |
| MERGE  | :40, :10 | 9:40, 10:10, 10:40 |

Each invocation:
1. Runs the agent's `find-work.sh` to identify actionable items
2. If work found: starts a new agent session per task
3. If no work: exits cleanly (no token spend beyond the script)

---

## Escalation

When an agent exhausts its retry limit (3 attempts):

1. Label the issue/PR `escalated`
2. @-mention the human (George) in an issue comment with:
   - What was attempted
   - Why it failed (error logs, screenshots)
   - What the agent thinks the human should do
3. Send a Telegram notification with a link to the issue

---

## Build Order

Agents will be built and tested one at a time:

1. **ONBOARD** — Required first; enables all other agents to operate
2. **PLAN** — Produces the specs and tasks that drive everything else
3. **BUILD** — Core value; can be tested with manually-created issues
4. **TEST** — Validates BUILD output
5. **REVIEW** — Quality gate
6. **MERGE** — Completes the loop

---

## Open Items

These will be tracked as GitHub issues for individual research and resolution:

1. **PLAN skill selection** — Evaluate deep-project, deep-plan, and other planning methodologies for the PLAN agent prompt
2. **BUILD skill selection** — Evaluate Claude Code coding skills/patterns for the BUILD agent prompt
3. **REVIEW skill selection** — Evaluate proven code review skills/prompts for the REVIEW agent
4. **TEST skill selection** — Evaluate testing methodologies and Playwright integration patterns
5. **MERGE conflict resolution strategy** — Define how MERGE tracks and resolves conflicts
6. **Cross-repo agent memory** — Determine approach for agents building institutional knowledge across repos (mem0 vs repo-local vs hybrid)
7. **Parallelization strategy** — Future: multiple BUILD agents or subagent spawning for parallel task execution

---

## Future Work

- Model economization (sonnet for simpler agents once stable)
- Parallel task execution within a spec
- Cross-repo learning (patterns from one repo inform work on another)
- Metrics dashboard (cycle times, retry rates, escalation frequency)
- Self-improvement loop (agents analyze their own failure patterns)
