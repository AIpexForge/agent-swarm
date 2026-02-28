# PRD: ONBOARD Capability for Blueprint

**Author:** Blueprint (AI) + George Sapp
**Date:** 2026-02-27
**Status:** Draft
**Version:** 1.1
**Target Repo:** AIpexForge/agent-swarm
**Taskmaster Optimized:** Yes

---

## Executive Summary

The ONBOARD capability enables Blueprint to prepare any GitHub repository for agent-swarm operation. When a user says "onboard <repo_name>", Blueprint loads the ONBOARD prompt, performs a deep codebase analysis, generates all required scaffolding (`.agents/commands.yml`, `AGENTS.md`, directory structure), creates workflow labels on GitHub, and opens a PR for human review. This is build order #1 — every other agent in the swarm depends on ONBOARD's output to function.

## Problem Statement

### Current Situation
The agent-swarm system requires specific repo scaffolding (`.agents/commands.yml`, `AGENTS.md`, `specs/`, `plans/`, workflow labels) before any agent can operate on a repository. Currently this scaffolding does not exist on any target repo, and there is no automated way to produce it.

### User Impact
Without ONBOARD, every repo must be manually scaffolded before any agent can pick up work. This is error-prone (missed fields in `commands.yml`, wrong directory structure, forgotten labels) and creates a cold-start barrier for every new repo.

### Business Impact
ONBOARD is a hard dependency for the entire agent-swarm pipeline. Until it exists, no other agent (BUILD, TEST, REVIEW, MERGE) can be deployed against any repo. It is the critical path blocker for the entire system.

### Why Now
ONBOARD is build order #1 in the agent-swarm roadmap. The spec and plan for the overall system are complete. This is the next deliverable.

---

## Goals & Success Metrics

| Goal | Metric | Baseline | Target | Timeframe | Measurement |
|------|--------|----------|--------|-----------|-------------|
| Accurate stack detection | % of commands.yml fields correctly inferred | 0% (manual) | ≥90% without human input | Per onboarding | Human review of PR |
| Complete scaffolding | All required files/dirs created per spec | 0 repos onboarded | Any repo onboardable | Per onboarding | PR contains all artifacts |
| Minimal human effort | Questions asked to human during onboarding | N/A (fully manual) | ≤3 questions per repo | Per onboarding | Count during session |
| Label setup | All 6 workflow labels created on target repo | 0 | 6 | Per onboarding | `gh label list` verification |

---

## User Stories

### US-001: Onboard a Local Repo
- **As a** developer using agent-swarm
- **I want to** say "onboard myproject" to Blueprint
- **So that** the repo is fully prepared for agent-swarm operation without manual scaffolding
- **Acceptance Criteria:**
  1. Blueprint loads the ONBOARD prompt and begins codebase analysis
  2. Blueprint scans the full codebase and infers stack, frameworks, commands, and conventions
  3. Blueprint generates `.agents/commands.yml`, `AGENTS.md`, `specs/`, `plans/`
  4. Blueprint creates the 6 workflow labels on the GitHub remote
  5. Blueprint opens a PR with all scaffolding on branch `agent-swarm-onboarding`
  6. Blueprint notifies the user with a summary and next steps
- **Task Hints:** Deep file scanning (2h), commands.yml generation (1h), AGENTS.md generation (1h), PR creation (30m)
- **Dependencies:** REQ-001, REQ-002, REQ-003, REQ-004, REQ-005

### US-002: Onboard a Remote Repo Not Yet Cloned
- **As a** developer
- **I want to** say "onboard myproject https://github.com/org/myproject.git" to Blueprint
- **So that** Blueprint clones the repo first, then onboards it
- **Acceptance Criteria:**
  1. Blueprint clones the repo to the local filesystem
  2. Onboarding proceeds identically to US-001
  3. Clone location follows existing workspace conventions
- **Task Hints:** Clone logic (30m), path resolution (30m)
- **Dependencies:** REQ-001

### US-003: Re-onboard a Previously Onboarded Repo
- **As a** developer
- **I want to** re-onboard a repo that already has `.agents/` or `specs/`
- **So that** the scaffolding is updated without losing intentional customizations
- **Acceptance Criteria:**
  1. Blueprint detects existing `.agents/` or `specs/` directories
  2. Blueprint warns the user and asks permission before proceeding
  3. If permitted, Blueprint reads existing files, determines what needs updating, and generates appropriate changes
  4. PR diff shows only meaningful changes, not unnecessary rewrites
- **Task Hints:** Existing file detection (30m), diff-aware generation (1h), user confirmation flow (30m)
- **Dependencies:** REQ-001, REQ-006

---

## Functional Requirements

### P0 — Must Have

#### REQ-001: Deep Codebase Scan
- **Description:** ONBOARD performs a comprehensive analysis of the target repository to infer stack, frameworks, database, key directories, conventions, test patterns, and deployment setup. This goes beyond reading manifest files — it examines actual source code, configuration files, directory structure, CI configs, Dockerfiles, and documentation.
- **Acceptance Criteria:**
  1. Reads and analyzes: package.json/Gemfile/requirements.txt/go.mod/Cargo.toml (and equivalents)
  2. Examines actual source files to detect frameworks (e.g., distinguishes Next.js from Express, Rails from Sinatra)
  3. Detects database type from connection strings, ORM configs, migration files, or docker-compose
  4. Identifies test framework and test file patterns (location, naming conventions)
  5. Reads CI configuration (.github/workflows, Jenkinsfile, etc.) for build/test/deploy commands
  6. Reads Dockerfile/docker-compose for environment setup
  7. Identifies the primary source directory, test directory, and documentation directory
  8. Detects code conventions (linting config, formatting config, TypeScript vs JavaScript, etc.)
  9. If monorepo detected (multiple package.json/go.mod at different levels, or workspace config): warn user, identify and onboard the primary stack only. Document other stacks as notes in AGENTS.md.
- **testStrategy:**
  - Acceptance criteria: Run ONBOARD against 3 diverse repos (Node.js/Next.js, Python/Django, Go) and verify each `commands.yml` field is correctly populated
  - Verification approach: Compare generated `commands.yml` values against fixture files checked into `test/fixtures/<repo>/expected-commands.yml`. All required fields must match exactly; optional fields may differ with documented rationale.
  - Edge cases to test: Repo with no tests, repo with non-standard directory layout, repo with multiple package managers. Monorepo triggers warning per REQ-001 AC9.
  - Performance threshold: Sampling-based scan completes in under 60 seconds for repos up to 10,000 files (see Scan Strategy below)
- **Scan Strategy:** ONBOARD does NOT read every file. It samples strategically: (1) all manifest/config files (package.json, Dockerfile, CI configs, etc.), (2) directory listing via `find`/`ls` for structure mapping, (3) representative source files — entry points, index files, up to 5 files per major directory. Skips: `node_modules/`, `vendor/`, `dist/`, `.git/`, build artifacts.
- **Technical Spec:** The scan reads files using the `read` tool and `exec` for directory traversal. No external APIs beyond the filesystem.
- **Task Breakdown:** File discovery script (1h), manifest parser (1h), source code analyzer (2h), CI config parser (1h), aggregation logic (1h)
- **Dependencies:** None

#### REQ-002: Generate `.agents/commands.yml`
- **Description:** Produce a `commands.yml` file that captures all commands an agent needs to build, test, lint, and run the project, plus environment metadata.
- **Acceptance Criteria:**
  1. File is valid YAML
  2. Contains `stack` field identifying the primary framework/stack
  3. Contains `commands` block with: `install`, `build`, `test`, `dev`, `lint` (each may be blank with a comment if not determinable)
  4. Contains `environment` block with: `port`, runtime version, env file reference
  5. Contains `paths` block with: `source`, `tests`, `docs` directories
  6. Every inferred value has a comment explaining the inference source (e.g., `# from package.json scripts.test`)
  7. Any field that could not be determined is left blank with a `# UNKNOWN — could not determine from codebase` comment
- **Schema:** Required top-level keys: `stack` (string), `commands` (map with keys: `install`, `build`, `test`, `dev`, `lint` — each string or null), `environment` (map with keys: `port`, runtime version field, `env_file` — each string or null), `paths` (map with keys: `source`, `tests`, `docs` — each string or null). Custom keys are permitted and preserved during re-onboarding.
- **testStrategy:**
  - Acceptance criteria: Generated YAML is parseable, conforms to schema above, all required keys present, comments trace inference sources
  - Verification approach: Parse generated YAML with a YAML parser, validate against schema. For command verification: confirm command strings are syntactically valid (non-empty, reference real binaries/scripts). Do NOT execute commands against arbitrary repos — validate syntax only.
  - Edge cases to test: Repo with no test command (field should be null with UNKNOWN comment), repo with multiple build targets, repo where dev server requires env vars
- **Technical Spec:**
  ```yaml
  # .agents/commands.yml
  stack: nextjs  # inferred from next.config.js

  commands:
    install: npm install          # from package.json
    build: npm run build          # from package.json scripts.build
    test: npm test                # from package.json scripts.test
    dev: npm run dev              # from package.json scripts.dev
    lint: npm run lint            # from package.json scripts.lint

  environment:
    port: 3000                    # from next.config.js / package.json scripts.dev
    node_version: "20"            # from .nvmrc
    env_file: .env.local          # detected .env.local in repo root

  paths:
    source: src/                  # primary source directory
    tests: __tests__/             # test file location
    docs: docs/                   # documentation directory
  ```
- **Task Breakdown:** YAML template (30m), field population from scan results (1h), comment generation (30m), validation (30m)
- **Dependencies:** REQ-001

#### REQ-003: Generate `AGENTS.md`
- **Description:** Produce an `AGENTS.md` file for the target repo that gives any cold-start agent comprehensive context about the project — conventions, architecture, key files, gotchas, and a reference to `commands.yml`.
- **Acceptance Criteria:**
  1. References `.agents/commands.yml` for build/test/run commands
  2. Describes the project's purpose (inferred from README or repo description)
  3. Lists key architectural patterns and conventions detected
  4. Documents important files and directories with brief descriptions
  5. Notes any non-obvious setup requirements (env vars, external services, database setup)
  6. Includes a "Conventions" section covering code style, commit message format, branching strategy (if detectable)
  7. Quality bar: a developer unfamiliar with the repo should be able to start contributing after reading only this file and `commands.yml`
- **testStrategy:**
  - Acceptance criteria: File contains all required sections, references are accurate, no hallucinated paths or commands
  - Verification approach: Cross-reference every path and command mentioned in AGENTS.md against actual repo contents. 100% of referenced paths must resolve to existing files or directories. Commands must match keys in commands.yml.
  - Edge cases to test: Repo with no README, repo with extensive existing documentation, repo with unconventional structure
- **Task Breakdown:** Template structure (30m), content generation from scan (1.5h), cross-reference validation (30m)
- **Dependencies:** REQ-001, REQ-002

#### REQ-004: Create Directory Structure
- **Description:** Create `specs/`, `specs/archive/`, and `plans/` directories in the target repo if they don't already exist.
- **Acceptance Criteria:**
  1. `specs/` directory exists with a `.gitkeep` file
  2. `specs/archive/` directory exists with a `.gitkeep` file
  3. `plans/` directory exists with a `.gitkeep` file
  4. Existing directories are not modified or overwritten
- **testStrategy:**
  - Acceptance criteria: All three directories exist in the PR, no existing files disturbed
  - Verification approach: Check PR diff for directory creation, verify no deletions of existing files
  - Edge cases to test: Repo that already has a `specs/` directory with content
- **Task Breakdown:** Directory creation (15m), existing directory detection (15m)
- **Dependencies:** None

#### REQ-005: Create Workflow Labels
- **Description:** Create the 6 core workflow labels on the target repo's GitHub remote so that other agents can apply them.
- **Acceptance Criteria:**
  1. The following labels are created: `plan:draft`, `ready-for-build`, `ready-for-test`, `ready-for-review`, `ready-for-merge`, `escalated`
  2. Labels have consistent, distinguishable colors
  3. Labels have brief descriptions explaining their purpose
  4. If a label already exists, it is skipped (not duplicated or errored)
  5. Label creation happens via `gh label create` CLI
  6. If label creation fails (e.g., insufficient permissions), warn the user listing which labels were not created and include manual creation instructions in the PR body
  7. Post-creation verification: run `gh label list` and confirm all 6 labels exist before proceeding
- **testStrategy:**
  - Acceptance criteria: `gh label list` on the repo shows all 6 labels with correct names, colors, and descriptions
  - Verification approach: Run `gh label list --repo <target>` after onboarding and verify output
  - Edge cases to test: Repo that already has some of the labels, repo where user lacks label-create permissions
  - Performance threshold: N/A
- **Technical Spec:**
  ```bash
  gh label create "plan:draft" --color "0E8A16" --description "Plan+spec PR under review" --repo <target> --force
  gh label create "ready-for-build" --color "1D76DB" --description "Task/PR ready for BUILD agent" --repo <target> --force
  gh label create "ready-for-test" --color "FBCA04" --description "PR ready for TEST agent" --repo <target> --force
  gh label create "ready-for-review" --color "D93F0B" --description "PR ready for REVIEW agent" --repo <target> --force
  gh label create "ready-for-merge" --color "0075CA" --description "PR ready for MERGE agent" --repo <target> --force
  gh label create "escalated" --color "B60205" --description "Human intervention required" --repo <target> --force
  ```
- **Task Breakdown:** Label definitions (15m), creation script (30m), idempotency check (15m)
- **Dependencies:** None

#### REQ-006: Handle Existing Scaffolding
- **Description:** When a target repo already has `.agents/` or `specs/` directories, ONBOARD must warn the user and ask permission before making changes. If permitted, it reads existing files, determines what needs updating, and generates only meaningful changes.
- **Merge Strategy:**
  - **commands.yml:** Key-level merge. Preserve existing custom keys not in ONBOARD's schema. For schema keys: overwrite inferred values only if they differ from existing. Leave human-edited values (those without `# UNKNOWN` comments) intact unless the new scan produces a higher-confidence inference. Add new keys with inference comments.
  - **AGENTS.md:** Section-level merge. Update sections that ONBOARD manages (project description, architecture, conventions, paths). Preserve any sections not in ONBOARD's template (human-added content). Append a "Last Updated" timestamp.
  - **Show diff summary to user before committing** — list what will change and what will be preserved.
- **Acceptance Criteria:**
  1. ONBOARD detects existing `.agents/` and/or `specs/` directories before writing
  2. If found, ONBOARD warns the user with a summary of what exists
  3. ONBOARD waits for explicit user permission before proceeding
  4. If permitted, ONBOARD reads existing files and applies the merge strategy above
  5. PR contains only meaningful changes (not full rewrites of unchanged content)
  6. Custom fields in commands.yml and human-added sections in AGENTS.md are preserved
  7. If the user denies permission, ONBOARD aborts cleanly with a message
- **testStrategy:**
  - Acceptance criteria: Warning is shown, permission is requested, diff is minimal, custom content preserved
  - Verification approach: Run ONBOARD against a repo with existing `.agents/commands.yml` containing custom fields (e.g., `deploy: kubectl apply`). Verify PR diff: (a) fields identical to inferred values are absent from diff, (b) differing fields appear with new values, (c) custom fields not in ONBOARD's schema are unchanged in output.
  - Edge cases to test: Existing `commands.yml` with custom fields not in the template, existing `AGENTS.md` with manual additions, partially corrupted prior onboarding (missing required keys)
- **Task Breakdown:** Detection logic (30m), user prompt flow (30m), diff-aware generation (1h)
- **Dependencies:** REQ-001

#### REQ-007: Open PR with Scaffolding
- **Description:** ONBOARD opens a single PR on the target repo containing all generated files, with a descriptive body that includes next steps for the human.
- **Acceptance Criteria:**
  1. PR is opened on branch `agent-swarm-onboarding` (or repo convention equivalent)
  2. PR targets the repo's default branch (usually `main`)
  3. PR title: `chore: onboard repo for agent-swarm`
  4. PR body contains:
     - Summary of what was detected (stack, frameworks, key decisions)
     - List of files created/modified
     - Any fields left as UNKNOWN with explanation
     - Next steps checklist (review commands.yml accuracy, verify AGENTS.md, etc.)
     - Any labels that failed to create (with manual instructions)
  5. PR is created via `gh pr create`
  6. If branch `agent-swarm-onboarding` already exists, delete it and recreate
  7. **Rollback:** If any step after branch creation fails (commit, push, or PR creation), delete the remote branch via `git push origin --delete agent-swarm-onboarding` and notify the user with the specific error
  8. If `gh` auth fails or user lacks push access, surface actionable error: "gh auth status shows [X]. Run `gh auth login` to authenticate, or verify you have push access to [repo]."
- **testStrategy:**
  - Acceptance criteria: PR exists on GitHub with correct branch, title, body sections, and all files. On failure, no dangling branches remain.
  - Verification approach: `gh pr view` the created PR and verify structure; check out the branch and verify file contents. For rollback: simulate a push failure and verify branch is cleaned up.
  - Edge cases to test: Repo with branch protection rules, repo where `agent-swarm-onboarding` branch already exists, auth failure mid-flow
- **Task Breakdown:** Branch creation (15m), commit logic (15m), PR body template (30m), `gh pr create` (15m)
- **Dependencies:** REQ-002, REQ-003, REQ-004

#### REQ-008: Human Fallback for Unknowns
- **Description:** After exhaustive inference effort, ONBOARD asks the human about any values it could not determine. Questions are batched, specific, and explain what was attempted.
- **Acceptance Criteria:**
  1. ONBOARD only asks after making maximum inference effort (file scanning, CI analysis, convention detection)
  2. Questions are batched into a single message, not asked one at a time
  3. Each question explains what ONBOARD tried and why it couldn't determine the value
  4. If the human says "skip" or "I don't know," the field is left as UNKNOWN in the output
  5. No more than one round of questions per onboarding (ask everything at once)
  6. Maximum 7 questions per batch. If more than 7 unknowns exist, make best-guess assumptions for the remainder and document them in AGENTS.md under an "Assumptions (auto-inferred, verify)" section
  7. Questions are formatted as a numbered list, each with: field name, what was tried, why it failed, and a suggested default if applicable
- **testStrategy:**
  - Acceptance criteria: Questions are batched, explain inference attempts, accept "skip" answers gracefully, max 7 questions
  - Verification approach: Run ONBOARD against a minimal repo (no CI, no README, ambiguous stack). Verify: (a) questions appear as a single numbered list, (b) each question includes field name + inference attempt explanation, (c) responding "skip" results in `# UNKNOWN — skipped by user` comment in output, (d) no more than 7 questions even if more unknowns exist
  - Edge cases to test: Repo where everything is determinable (0 questions), repo with 10+ unknowns (verify cap at 7 with remaining as documented assumptions)
- **Task Breakdown:** Unknown collection logic (30m), question formatting (30m), response handling (30m)
- **Dependencies:** REQ-001

#### REQ-009: ONBOARD Prompt File
- **Description:** The ONBOARD capability is defined in a separate prompt file at `agents/onboard/AGENTS.md` in the agent-swarm repo. Blueprint's main prompt references this file and loads it when the user triggers onboarding.
- **Acceptance Criteria:**
  1. Prompt file exists at `agents/onboard/AGENTS.md`
  2. Prompt contains complete instructions for the ONBOARD workflow (scan → generate → ask → PR → notify)
  3. Blueprint's main prompt contains a reference to load this file on "onboard" trigger
  4. Prompt is self-contained — an agent reading only this file can execute the full ONBOARD workflow
- **testStrategy:**
  - Acceptance criteria: File exists, is complete, Blueprint can load and execute it
  - Verification approach: (1) Verify file exists at `agents/onboard/AGENTS.md`, (2) Verify prompt contains sections for each ONBOARD phase (scan, generate, ask, PR, notify), (3) Load prompt in a test session, send "onboard <test-repo>", verify session produces a PR URL in its final output message
  - Edge cases to test: Prompt file missing (Blueprint should error with clear message)
- **Task Breakdown:** Prompt writing (2h), Blueprint reference integration (30m)
- **Dependencies:** All other REQs (prompt must describe the full workflow)

### P1 — Should Have

#### REQ-010: Clone Remote Repos
- **Description:** If the user provides a git remote URL alongside the repo name, ONBOARD clones the repo before beginning analysis.
- **Acceptance Criteria:**
  1. Accepts `onboard <name> <git-url>` syntax
  2. Clones to a consistent workspace location
  3. Handles clone failures gracefully with specific error messages:
     - Auth failure: "Clone failed — private repo requires authentication. Run `gh auth login` or verify SSH keys."
     - Not found: "Repository not found at <URL>. Verify the URL and try again."
     - Network error: "Clone failed — network error. Check connectivity and retry."
  4. If repo already exists locally, uses the local copy (does not re-clone)
  5. On any clone failure, clean up partial directory (if created) and abort onboarding with clear message
- **testStrategy:**
  - Acceptance criteria: Repo is cloned, onboarding proceeds normally. On failure, no partial directories remain.
  - Verification approach: Provide a public repo URL, verify clone + full onboarding flow. For failure: provide invalid URL, verify error message and no leftover directories.
  - Edge cases to test: Private repo requiring auth (verify auth error message), URL with typo (verify not-found message), repo already cloned locally (verify skip-clone behavior)
- **Task Breakdown:** Clone logic (30m), path resolution (30m), error handling (30m)
- **Dependencies:** REQ-001

#### REQ-011: User Notification on Completion
- **Description:** After opening the PR, ONBOARD sends the user a summary via the active channel (Telegram) with key findings, the PR link, and next steps.
- **Acceptance Criteria:**
  1. Message includes: detected stack, number of files created, PR link
  2. Message includes next steps: review the PR, merge, then repo is ready for planning
  3. Message is concise (under 20 lines)
- **testStrategy:**
  - Acceptance criteria: Notification sent with all required fields
  - Verification approach: Verify message content after test onboarding
- **Task Breakdown:** Message template (15m), integration (15m)
- **Dependencies:** REQ-007

---

## Non-Functional Requirements

- **Performance:** Sampling-based codebase scan completes in under 60 seconds for repos up to 10,000 files (scans manifests, configs, CI files, and representative source files — not exhaustive). Entire onboarding flow completes in under 5 minutes.
- **Security:** ONBOARD does not execute any code from the target repo. It only reads files and creates GitHub resources (labels, PR). No secrets are written to generated files.
- **Reliability:** If any step fails (scan, PR creation, label creation), ONBOARD reports the failure clearly and does not leave partial state (no half-created PRs, no dangling branches). See REQ-007 rollback strategy.
- **Compatibility:** Works with any GitHub-hosted repo. Supports Node.js, Python, Go, Ruby, Rust, and Java/Kotlin ecosystems for stack detection. Unknown stacks are flagged, not errored.
- **Prerequisites:** Assumes `gh` CLI is installed and authenticated with access to the target repo. Private repo access errors surface a human-readable failure message (see REQ-010).

---

## Technical Considerations

### System Architecture

```
User: "onboard myproject"
  │
  ▼
Blueprint (main prompt)
  │ loads agents/onboard/AGENTS.md
  ▼
ONBOARD Capability
  │
  ├─ 1. Locate or clone repo
  ├─ 2. Deep codebase scan (read files, exec for traversal)
  ├─ 3. Generate commands.yml + AGENTS.md
  ├─ 4. Detect existing scaffolding → warn if present
  ├─ 5. Ask human about unknowns (if any)
  ├─ 6. Create specs/, plans/ directories
  ├─ 7. Create GitHub labels (gh label create)
  ├─ 8. Commit, push, open PR (gh pr create)
  └─ 9. Notify user with summary + next steps
```

### Tools Used
- `read` — File content analysis
- `exec` — Directory traversal (`find`, `ls`), git operations, `gh` CLI for labels and PR
- Telegram messaging — User interaction (questions, notifications)

### No External Dependencies
ONBOARD uses only the filesystem and GitHub CLI. No external APIs, no databases, no additional tools.

---

## Implementation Roadmap

1. **Phase 1: Prompt file** — Write `agents/onboard/AGENTS.md` with full ONBOARD instructions
2. **Phase 2: Blueprint integration** — Add ONBOARD trigger detection to Blueprint's main prompt
3. **Phase 3: Test against real repos** — Run ONBOARD against 2-3 diverse repos, validate output quality

---

## Dependency Chain

```
REQ-001 (scan) → REQ-002 (commands.yml) → REQ-003 (AGENTS.md) → REQ-007 (PR)
REQ-001 (scan) → REQ-006 (existing scaffolding)
REQ-001 (scan) → REQ-008 (human fallback)
REQ-004 (directories) → REQ-007 (PR)
REQ-005 (labels) — independent, can run in parallel
REQ-009 (prompt file) — depends on all others being defined
REQ-010 (clone) → REQ-001 (scan)
REQ-007 (PR) → REQ-011 (notification)
```

---

## Out of Scope

- **Cron-driven re-onboarding** — ONBOARD is manual-trigger only
- **Multi-repo batch onboarding** — One repo at a time
- **CI/CD setup** — ONBOARD does not create GitHub Actions or deployment pipelines
- **Agent deployment** — ONBOARD scaffolds the repo; deploying agents against it is a separate concern
- **Non-GitHub hosting** — GitLab, Bitbucket, etc. are not supported in v1
- **`config/repos.yml` management** — Removed from system; agents discover repos on filesystem
- **Full monorepo support** — ONBOARD detects monorepos and warns the user, but only onboards the primary stack. Per-package commands.yml is not supported in v1.

---

## Open Questions & Risks

| # | Question/Risk | Owner | Status |
|---|---------------|-------|--------|
| 1 | What workspace directory should cloned repos be placed in? | George | Open |
| 2 | Should ONBOARD detect and document deployment targets (Vercel, AWS, etc.)? | Blueprint | Open |
| 3 | ~~Monorepo handling~~ — Resolved: detect and warn, onboard primary stack only (see REQ-001 AC9, Out of Scope) | Blueprint | Resolved |
| 4 | Branch protection rules may prevent PR creation — fallback strategy? | Blueprint | Open |

---

## Appendix: Task Breakdown Hints

| Task | Size | Dependencies |
|------|------|-------------|
| Deep codebase scanner (multi-ecosystem) | L (6h) | None |
| commands.yml generator with inference comments | M (2.5h) | Scanner |
| AGENTS.md generator | M (2h) | Scanner, commands.yml |
| Directory scaffolding (specs, plans) | S (30m) | None |
| GitHub label creation script | S (1h) | None |
| Existing scaffolding detection + user prompt | M (2h) | Scanner |
| PR creation (branch, commit, body template) | S (1.5h) | All generators |
| Human fallback question batching | S (1.5h) | Scanner |
| ONBOARD prompt file (agents/onboard/AGENTS.md) | L (3h) | All above defined |
| Blueprint trigger integration | S (30m) | Prompt file |
| Clone logic for remote repos | S (1.5h) | None |
| Completion notification | S (30m) | PR creation |
