# Oh My OpenCode — Agent Prompts (Adapted for OpenClaw)

System prompts extracted from [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) (dev branch, 2026-03-04), adapted to use OpenClaw's tooling and agent-swarm's GitHub-based state model.

## Key Adaptations from OMO → OpenClaw

| OMO Concept | OpenClaw Equivalent |
|-------------|-------------------|
| `task(subagent_type=X)` | `sessions_spawn(agentId=X)` |
| `task(category=X, load_skills=[...])` | `sessions_spawn` with model/task context |
| `run_in_background=true` | `sessions_spawn(mode="run")` — push-based, auto-announces completion |
| `background_output(task_id)` | Sub-agent auto-announces; use `sessions_history` if needed |
| `session_id` for resume | `sessions_send(sessionKey, message)` to existing session |
| `.sisyphus/plans/*.md` | GitHub issues with labels (`plan:draft`, `ready-for-build`, etc.) |
| `.sisyphus/notepads/` | GitHub issue comments (structured, searchable) |
| `todowrite` / `TaskCreate` | GitHub issue checkboxes or agent comments |
| `lsp_diagnostics` | `exec` running project's lint/typecheck commands |
| `Question` tool | Telegram inline buttons via `message(action=send, buttons=[...])` |
| `call_omo_agent` | `sessions_spawn` or `sessions_send` |
| OpenCode hooks | OpenClaw cron jobs + find-work.sh pre-checks |

## Agent Roster

| Agent | Role | File | Relevance to agent-swarm |
|-------|------|------|------------------------|
| **Sisyphus** | Main orchestrator | [sisyphus.md](sisyphus.md) | Patterns for BUILD agent |
| **Hephaestus** | Autonomous deep worker | [hephaestus.md](hephaestus.md) | Patterns for BUILD agent |
| **Prometheus** | Strategic planner | [prometheus.md](prometheus.md) | Patterns for PLAN/Blueprint agent |
| **Atlas** | Plan executor/conductor | [atlas.md](atlas.md) | Patterns for BUILD agent orchestration |
| **Oracle** | Architecture consultant | [oracle.md](oracle.md) | Reusable as sub-agent for any agent |
| **Metis** | Pre-planning gap analyzer | [metis.md](metis.md) | Sub-agent for PLAN agent |
| **Momus** | Plan reviewer | [momus.md](momus.md) | Sub-agent for REVIEW agent |
| **Explore** | Codebase grep | [explore.md](explore.md) | Sub-agent for BUILD/PLAN |
| **Librarian** | External docs search | [librarian.md](librarian.md) | Sub-agent for BUILD/PLAN |
| **Multimodal Looker** | Vision/PDF analysis | [multimodal-looker.md](multimodal-looker.md) | Utility sub-agent |

## Notes

- These are reference prompts, not direct agent configurations. The patterns and prompt techniques
  should be extracted and incorporated into each agent-swarm agent's AGENTS.md / PROMPT.md.
- agent-swarm uses GitHub (issues, PRs, labels, comments) as shared state instead of local files.
- Sub-agents in OpenClaw are push-based (they auto-announce completion) — no polling needed.
