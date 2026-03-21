# ONBOARD — Blueprint Capability Prompt

You are executing the **ONBOARD** capability. Your job: analyze a target repository and produce all scaffolding required for agent-swarm to operate on it.

This is a one-shot flow. Scan → generate → ask → commit → PR → notify → done.

---

## Input

You receive:
- `repo_name`: Name of the target repo
- `repo_path`: Local filesystem path to the repo (resolved by Blueprint before loading this prompt)
- `repo_remote`: GitHub remote URL (e.g., `AIpexForge/snaphappy` or full URL) — for label creation and PR

If the repo is not yet cloned, Blueprint handles cloning before invoking this prompt.

---

## Phase 1: Pre-Flight Checks

Before scanning:

1. **Verify repo exists** at `repo_path`. If not, abort: "Repo not found at `<path>`. Verify the path and try again."

2. **Verify GitHub access:**
   ```bash
   gh repo view <repo_remote> --json name -q .name
   ```
   If this fails, abort with actionable error:
   - Auth failure → "gh auth status shows no valid token. Run `gh auth login`."
   - Not found → "Repository `<remote>` not found. Verify the name/URL."

3. **Check for existing scaffolding:**
   - If `.agents/` or `specs/` directories exist:
     - Warn the user: "This repo already has agent-swarm scaffolding: [list what exists]"
     - Ask: "Proceed with re-onboarding? I'll preserve your customizations and only update what's changed."
     - If user declines → abort cleanly: "Onboarding cancelled."
     - If user approves → set `RE_ONBOARD=true`, read all existing files for merge

4. **Check for existing branch:**
   ```bash
   git ls-remote --heads origin agent-swarm-onboarding
   ```
   If branch exists, delete it:
   ```bash
   git push origin --delete agent-swarm-onboarding
   ```

---

## Phase 2: Codebase Scan

**Strategy: Sample, don't exhaust.** You are reading representative files, not every file in the repo.

### Step 2.1 — Structure Map
```bash
# Directory structure (excluding noise)
find <repo_path> -type f \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/vendor/*" \
  -not -path "*/dist/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/build/*" \
  -not -path "*/.next/*" \
  -not -path "*/target/*" \
  | head -500

# Top-level directory listing
ls -la <repo_path>
```

### Step 2.2 — Manifest & Config Files

Read ALL of these that exist (they are small and high-signal):

| Category | Files |
|----------|-------|
| **Package manifests** | `package.json`, `Gemfile`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `Makefile` |
| **Lock files** (skim, don't deep-read) | `package-lock.json` (just check it exists), `yarn.lock`, `pnpm-lock.yaml`, `Pipfile.lock`, `go.sum` |
| **Runtime config** | `.nvmrc`, `.node-version`, `.python-version`, `.ruby-version`, `.tool-versions`, `runtime.txt` |
| **CI/CD** | `.github/workflows/*.yml`, `Jenkinsfile`, `.gitlab-ci.yml`, `.circleci/config.yml`, `Dockerfile`, `docker-compose.yml`, `docker-compose.yaml` |
| **Linting/formatting** | `.eslintrc*`, `.prettierrc*`, `biome.json`, `.editorconfig`, `rustfmt.toml`, `.rubocop.yml`, `.flake8`, `setup.cfg`, `pyproject.toml [tool.*]` |
| **Environment** | `.env.example`, `.env.local`, `.env.template`, any `.env*` file (DO NOT read `.env` — it may contain secrets; just note its existence) |
| **Documentation** | `README.md`, `CONTRIBUTING.md`, `docs/` directory listing |
| **Existing scaffolding** | `.agents/commands.yml`, `AGENTS.md`, `CLAUDE.md`, `CURSOR.md` |

### Step 2.3 — Representative Source Files

Read entry points and up to 5 files per major source directory:

- **Entry points:** `src/index.*`, `src/main.*`, `src/app.*`, `app/page.*`, `main.*`, `cmd/*/main.go`, `lib/*.rb` (first file)
- **Config files in source:** `src/config.*`, `config/*`, `settings.*`
- **Test files:** Read 2-3 test files to understand:
  - Test framework (Jest, Vitest, pytest, Go testing, etc.)
  - File naming pattern (`*.test.ts`, `test_*.py`, `*_test.go`)
  - Assertion style (`expect().toBe()`, `assert`, `t.Run()`)
  - Setup/teardown patterns (beforeEach, fixtures, test helpers)
  - Whether tests are unit, integration, or e2e (or a mix)
  Record these findings for the `testing` section in `commands.yml` and the `Key Abstractions & Patterns` table in `AGENTS.md`.
- **Database:** Migration files (first and last), ORM config, `prisma/schema.prisma`, `db/schema.rb`, `alembic.ini`

### Step 2.4 — Monorepo Detection

Check for:
- `workspaces` in `package.json`
- `pnpm-workspace.yaml`
- `lerna.json`
- Multiple `package.json`/`go.mod`/`Cargo.toml` at different directory levels
- `nx.json`, `turbo.json`

If detected: warn user, identify primary package (usually the root or the one with the most source code), onboard that only. Document others in AGENTS.md notes.

### Step 2.5 — Inference Aggregation

After scanning, build an internal model:

```
stack: <primary framework/stack>
language: <primary language>
frameworks: [<detected frameworks>]
database: <type + ORM if applicable>
commands:
  install: <command or UNKNOWN>
  build: <command or UNKNOWN>
  test: <command or UNKNOWN>
  dev: <command or UNKNOWN>
  lint: <command or UNKNOWN>
environment:
  port: <number or UNKNOWN>
  runtime_version: <version or UNKNOWN>
  env_file: <path or UNKNOWN>
paths:
  source: <dir>
  tests: <dir or UNKNOWN>
  docs: <dir or UNKNOWN>
testing:
  framework: <detected test framework or UNKNOWN>
  runner: <test runner command or UNKNOWN>
  file_pattern: <glob pattern or UNKNOWN>
  assertion_style: <assertion pattern or UNKNOWN>
  location: <test directory or UNKNOWN>
  ci: <CI workflow that runs tests or UNKNOWN>
conventions:
  code_style: <linter/formatter>
  commit_format: <conventional commits, etc. or UNKNOWN>
  branching: <strategy or UNKNOWN>
patterns: [<list of {concern, pattern, location} detected from source scan>]
unknowns: [<list of fields that could not be determined>]
```

**Inference rules:**
- `package.json` scripts → commands (install=`npm install`, build=scripts.build, test=scripts.test, etc.)
- `Makefile` targets → commands
- CI workflow steps → commands (often more accurate than manifest scripts)
- `next.config.*` → stack is Next.js, not just Node
- `nuxt.config.*` → Nuxt
- `angular.json` → Angular
- `vite.config.*` → Vite
- Presence of `prisma/` → Prisma ORM
- Presence of `migrations/` + `alembic` → SQLAlchemy + Alembic
- `docker-compose.yml` services → database type (postgres, mysql, redis, etc.)

---

## Phase 3: Human Fallback

Review your `unknowns` list. If any unknowns remain:

1. **Cap at 7 questions.** If more than 7 unknowns exist, make your best guess for the extras and document them in AGENTS.md under "Assumptions (auto-inferred, verify)."

2. **Batch all questions in a single message.** Format:

```
I couldn't determine the following from the codebase:

1. **test command** — Checked package.json scripts, CI workflows, and Makefile. No test runner detected. What command runs tests? (or "skip")
2. **database type** — No docker-compose, no ORM config, no connection strings found. What database does this project use? (or "skip")
3. ...

Reply with answers or "skip" for any you'd like to leave as unknown.
```

3. **Process response:**
   - Direct answers → update the internal model
   - "skip" or "I don't know" → mark as `# UNKNOWN — skipped by user`
   - If user answers with new info, re-infer related fields (e.g., if they say "PostgreSQL", also infer common Postgres port 5432)

4. **If zero unknowns → skip this phase entirely.** Do not ask unnecessary questions.

---

## Phase 4: Generate Artifacts

### 4.1 — `.agents/commands.yml`

Generate the file with every field traced to its source:

```yaml
# .agents/commands.yml
# Generated by ONBOARD — verify and adjust as needed

stack: <stack>  # inferred from <source>

commands:
  install: <cmd>   # from <source>
  build: <cmd>     # from <source>
  test: <cmd>      # from <source>
  dev: <cmd>       # from <source>
  lint: <cmd>      # from <source>

environment:
  port: <port>              # from <source>
  <runtime>_version: "<v>"  # from <source>
  env_file: <path>          # detected <file> in repo

paths:
  source: <dir>/   # primary source directory
  tests: <dir>/    # test file location
  docs: <dir>/     # documentation directory

testing:
  framework: <framework>        # from <source> (e.g., vitest, jest, pytest, go test)
  runner: <runner>               # from <source> (e.g., vitest, jest, pytest, cargo test)
  file_pattern: "<glob>"         # from scanned test files (e.g., "**/*.test.ts", "test_*.py")
  assertion_style: <style>       # from sample test (e.g., expect/toBe, assert, t.Run)
  location: <dir>/               # from directory scan
  ci: <workflow_path>            # CI config that runs tests (e.g., .github/workflows/test.yml)
```

**If `RE_ONBOARD=true`:** Apply merge strategy:
- Read existing `commands.yml`
- Preserve custom keys not in the schema
- Overwrite schema keys only if the new value differs AND has higher confidence (traced to a more authoritative source)
- Preserve human-edited values (those without `# UNKNOWN` comments) unless new scan contradicts with evidence
- Show the user a diff summary before proceeding

### 4.2 — `AGENTS.md`

Generate a comprehensive context file:

```markdown
# AGENTS.md

> Auto-generated by ONBOARD. Verify accuracy and extend as needed.

## Project Overview
[2-3 sentences describing the project, inferred from README or repo description]

## Quick Reference
- **Stack:** <stack>
- **Language:** <language>
- **Framework(s):** <frameworks>
- **Database:** <db type + ORM>
- **Commands:** See `.agents/commands.yml`

## Architecture
[Key architectural patterns detected. Directory structure overview. Data flow if inferable.]

### Key Files & Directories
| Path | Purpose |
|------|---------|
| `src/` | Primary source code |
| `...` | ... |

### Key Abstractions & Patterns
[MANDATORY. For each common concern the codebase handles, document the pattern and location.
This table is used by reviewer sub-agents to detect duplication and pattern conflicts during planning.]

| Concern | Pattern | Location | Notes |
|---------|---------|----------|-------|
| Auth | [e.g., JWT middleware] | [e.g., src/middleware/auth.ts] | |
| Validation | [e.g., Zod schemas] | [e.g., src/validators/] | |
| Error handling | [e.g., Custom error class] | [e.g., src/lib/errors.ts] | |
| Database access | [e.g., Prisma ORM] | [e.g., prisma/schema.prisma] | |
| Events/messaging | [e.g., EventEmitter] | [e.g., src/events/] | |
| Logging | [e.g., Winston/Pino] | [e.g., src/lib/logger.ts] | |
| Config/env | [e.g., dotenv + typed config] | [e.g., src/config.ts] | |

Remove rows that don't apply. Add rows for patterns specific to this codebase.
If the project is too small to have established patterns, state: "Early-stage — no established patterns yet."

## Conventions
- **Code Style:** <linter/formatter + config file>
- **Commit Messages:** <format if detected, otherwise "Not detected — verify">
- **Branching:** <strategy if detected>
- **Testing:** <framework, file naming pattern, location>

## Setup
[Non-obvious setup requirements: env vars needed, external services, database initialization, etc.]

## Gotchas
[Anything unusual or non-obvious detected during scan — e.g., custom build steps, monorepo quirks, legacy patterns]

## Assumptions (auto-inferred, verify)
[Any values where ONBOARD made a best-guess assumption beyond the 7-question cap. Empty if all values were determined or asked.]

---
*Last onboarded: <date>*
```

**If `RE_ONBOARD=true`:** Apply merge strategy:
- Read existing `AGENTS.md`
- Update ONBOARD-managed sections (Overview, Quick Reference, Architecture, Key Files, Conventions)
- Preserve human-added sections not in the template
- Update "Last onboarded" timestamp
- Show diff summary to user

### 4.3 — Directory Structure

```bash
mkdir -p <repo_path>/specs/archive
mkdir -p <repo_path>/plans
touch <repo_path>/specs/.gitkeep
touch <repo_path>/specs/archive/.gitkeep
touch <repo_path>/plans/.gitkeep
```

Skip any directory that already exists (do not overwrite contents).

---

## Phase 5: Create GitHub Labels

Create the 6 workflow labels:

```bash
gh label create "plan:draft" --color "0E8A16" --description "Plan+spec PR under review" --repo <repo_remote> --force
gh label create "ready-for-build" --color "1D76DB" --description "Task/PR ready for BUILD agent" --repo <repo_remote> --force
gh label create "ready-for-test" --color "FBCA04" --description "PR ready for TEST agent" --repo <repo_remote> --force
gh label create "ready-for-review" --color "D93F0B" --description "PR ready for REVIEW agent" --repo <repo_remote> --force
gh label create "ready-for-merge" --color "0075CA" --description "PR ready for MERGE agent" --repo <repo_remote> --force
gh label create "escalated" --color "B60205" --description "Human intervention required" --repo <repo_remote> --force
```

**After creation, verify:**
```bash
gh label list --repo <repo_remote> --json name -q '.[].name' | grep -E "plan:draft|ready-for-build|ready-for-test|ready-for-review|ready-for-merge|escalated"
```

If any label failed to create, record it. Include manual creation instructions in the PR body.

---

## Phase 5b: Register Sub-Agents (if Blueprint is the planning agent)

If the system uses Blueprint as the planning agent, ensure the following sub-agents are registered in the user's `openclaw.json` under `agents.list`, and that Blueprint's `subagents.allowAgents` lists them.

**Required sub-agents:**

| agentId | Name | Model | Purpose |
|---------|------|-------|---------|
| `research` | Research Agent | sonnet | Technical question investigation |
| `quality-validator` | Quality Validator | sonnet | PRD scoring against checklist |
| `architecture-auditor` | Architecture Auditor | sonnet | Architecture soundness review |
| `integration-auditor` | Integration Auditor | sonnet | Codebase compatibility review |
| `contradiction-detector` | Contradiction Detector | sonnet | Internal inconsistency detection |
| `coherence-auditor` | Coherence Auditor | sonnet | Plan coherence review |
| `testability-auditor` | Testability Auditor | sonnet | testStrategy verifiability |
| `decompose` | Decompose Agent | sonnet | Task breakdown into GitHub issues |

**For each agent, add an entry to `openclaw.json` `agents.list`:**
```json
{
    "id": "<agentId>",
    "name": "<Name>",
    "agentDir": "<path-to-agent-swarm>/agents/blueprint/sub-agents/<agentId>",
    "model": "anthropic/claude-sonnet-4-6"
}
```

The `agentDir` points directly to the sub-agent's directory in the agent-swarm repo (under `agents/blueprint/sub-agents/`), where `AGENTS.md` is the prompt file loaded by OpenClaw.

**Then update Blueprint's entry to allow spawning them:**
```json
{
    "id": "blueprint",
    ...
    "subagents": {
        "model": "anthropic/claude-sonnet-4-6",
        "allowAgents": [
            "architecture-auditor",
            "coherence-auditor",
            "contradiction-detector",
            "integration-auditor",
            "testability-auditor",
            "quality-validator",
            "research",
            "decompose"
        ]
    }
}
```

**IMPORTANT:** The key is `allowAgents` (not `allowedAgents`). Without this, Blueprint defaults to only being able to spawn itself. Sub-agent spawns will silently fail.

**Skip this phase** if sub-agents are already registered (check `openclaw.json` first).

---

## Phase 6: Commit, Push, PR

### 6.1 — Branch & Commit

```bash
cd <repo_path>
git checkout -b agent-swarm-onboarding
git add .agents/commands.yml AGENTS.md specs/ plans/
git commit -m "chore: onboard repo for agent-swarm

Generated by ONBOARD capability:
- .agents/commands.yml (stack: <stack>)
- AGENTS.md (project context for cold-start agents)
- specs/ and plans/ directories
- 6 workflow labels created on GitHub"
```

### 6.2 — Push & PR

```bash
git push origin agent-swarm-onboarding
```

If push fails:
- Auth error → abort with: "Push failed — verify `gh auth status` and repo write access."
- Other error → abort with the specific error message
- In both cases: `git checkout main && git branch -D agent-swarm-onboarding` (local cleanup)

Create PR:

```bash
gh pr create \
  --title "chore: onboard repo for agent-swarm" \
  --label "plan:draft" \
  --body "<PR_BODY>"
```

### PR Body Template

```markdown
## Agent-Swarm Onboarding

ONBOARD analyzed this repository and generated the scaffolding needed for agent-swarm to operate.

### Detected Stack
- **Stack:** <stack>
- **Language:** <language>
- **Framework(s):** <frameworks>
- **Database:** <database>

### Files Created/Modified
- `.agents/commands.yml` — Build/test/run commands with inference sources
- `AGENTS.md` — Project context for cold-start agents
- `specs/` — Spec directory (with archive subdirectory)
- `plans/` — Plan directory

### Unknown Fields
[List any UNKNOWN fields with explanation, or "None — all fields determined"]

### Labels Created
[List all 6 labels, or note any that failed with manual instructions]

### ⚠️ Review Checklist
- [ ] Verify `.agents/commands.yml` — are the commands correct?
- [ ] Verify `AGENTS.md` — is the project description accurate?
- [ ] Check for any UNKNOWN fields and fill them in if possible
- [ ] Merge when satisfied — the repo will be ready for agent-swarm planning

### Failed Labels (if any)
[Instructions for manually creating any labels that failed]
```

### 6.3 — Rollback on Failure

If PR creation fails after push succeeded:
```bash
git push origin --delete agent-swarm-onboarding
git checkout main
git branch -D agent-swarm-onboarding
```
Notify user: "PR creation failed: [error]. Branch cleaned up. Try again or check `gh auth status`."

---

## Phase 7: Notify User

Send a concise summary:

```
✅ Onboarded <repo_name> — <stack> (<language>)

• .agents/commands.yml — <N> commands inferred
• AGENTS.md — project context generated
• 6 workflow labels created
• 8 sub-agents registered (or already present)
• <N> unknowns (see PR for details)

PR: <link>

Review the PR, verify the commands, and merge when ready. After merge, this repo is ready for planning.
```

**Done.** ONBOARD exits after notification.

---

## Error Handling Summary

| Error | Action |
|-------|--------|
| Repo not found on filesystem | Abort with path error |
| `gh` auth failure | Abort with `gh auth login` instructions |
| Repo not found on GitHub | Abort with URL verification message |
| Existing scaffolding, user declines | Abort cleanly |
| Branch already exists | Delete and recreate |
| Push failure | Local cleanup + abort with error |
| PR creation failure | Remote branch cleanup + abort with error |
| Label creation failure | Warn + include manual instructions in PR |
| >7 unknowns | Ask top 7, assume rest, document assumptions |

---

## Rules

1. **Never execute code from the target repo.** Read only. No `npm install`, no `make`, no `python setup.py`.
2. **Never read `.env` files.** They may contain secrets. Note their existence only.
3. **Every inference must be traceable.** Comment every value in commands.yml with its source.
4. **Don't hallucinate.** If you can't find evidence for a value, mark it UNKNOWN. Never guess framework versions, ports, or commands without evidence.
5. **Respect existing work.** During re-onboarding, preserve human customizations. When in doubt, keep the existing value.
6. **Be concise with the human.** Batch questions, cap at 7, accept "skip". Don't over-explain.
7. **Clean up after yourself.** If something fails, leave no dangling branches, no partial files.
