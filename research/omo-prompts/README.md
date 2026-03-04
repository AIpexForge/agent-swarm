# Oh My OpenCode — Agent Prompts (Extracted)

Raw system prompts extracted from [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) (dev branch, 2026-03-04).

These are the actual prompts that define each agent's behavior. The value proposition of OMO is ~80% prompt engineering, ~20% harness tooling (hash-anchored edits, LSP integration, category→model routing, background agent management).

## Agent Roster

| Agent | Role | File | Read-Only? | Cost |
|-------|------|------|-----------|------|
| **Sisyphus** | Main orchestrator | [sisyphus.md](sisyphus.md) | No | EXPENSIVE |
| **Hephaestus** | Autonomous deep worker (GPT-native) | [hephaestus.md](hephaestus.md) | No | EXPENSIVE |
| **Prometheus** | Strategic planner (interview → plan) | [prometheus.md](prometheus.md) | Write .md only | EXPENSIVE |
| **Atlas** | Plan executor/conductor | [atlas.md](atlas.md) | Reads + runs cmds | EXPENSIVE |
| **Oracle** | Architecture consultant | [oracle.md](oracle.md) | Read-only | EXPENSIVE |
| **Metis** | Pre-planning gap analyzer | [metis.md](metis.md) | Read-only | EXPENSIVE |
| **Momus** | Plan reviewer (approval-biased) | [momus.md](momus.md) | Read-only | EXPENSIVE |
| **Explore** | Codebase grep specialist | [explore.md](explore.md) | Read-only | FREE |
| **Librarian** | External docs/OSS search | [librarian.md](librarian.md) | Read-only | CHEAP |
| **Multimodal Looker** | Vision/PDF analysis | [multimodal-looker.md](multimodal-looker.md) | Read-only | CHEAP |

## Notes

- Sisyphus and Hephaestus prompts are **dynamically assembled** at runtime — sections like tool selection tables, delegation tables, and agent lists are injected based on what's available. The prompts here show the static template with `{PLACEHOLDER}` markers where dynamic content would go.
- Prometheus is assembled from 6 modular sections (identity, interview, plan-generation, high-accuracy, plan-template, behavioral-summary).
- Atlas has `{PLACEHOLDER}` markers for category/agent/skill sections injected at runtime.
- All agents use OpenCode's `task()` function for delegation — this is the harness's subagent spawning mechanism.
