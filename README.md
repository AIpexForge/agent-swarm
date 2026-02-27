# OpenClaw-Workflow

Decoupled AI agents for the software development lifecycle. Each agent has a single responsibility, communicates through GitHub primitives (issues, PRs, labels), and runs on its own schedule.

## Agents

| Agent | Purpose | Trigger |
|-------|---------|---------|
| **ONBOARD** | Prepare repos for agent operation | Manual |
| **PLAN** | Planning interviews → specs → task issues | Cron :00/:30 + interactive |
| **BUILD** | Write code, submit PRs | Cron :10/:40 |
| **TEST** | Automated + manual testing | Cron :20/:50 + deploy hooks |
| **REVIEW** | Code review for quality + spec compliance | Cron :30/:00 |
| **MERGE** | Merge PRs, manage feature branches | Cron :40/:10 |

## How It Works

```
User idea → PLAN → Spec PR → Human approves → Tasks → BUILD → TEST → REVIEW → MERGE
```

See [specs/FEAT-agent-workflow-system.md](specs/FEAT-agent-workflow-system.md) for the full specification.

## Status

🚧 In development — spec phase.
