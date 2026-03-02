# Taskmaster Methodology — Extracted Reference

> Source: [eyaltoledano/claude-task-master](https://github.com/eyaltoledano/claude-task-master)
> License: MIT
> Extracted: 2026-02-27 for Blueprint agent design

---

## Core Pipeline

```
PRD → parse-prd → tasks.json → analyze-complexity → expand → implement → update → done
```

## Key Prompts Extracted

### parse-prd (PRD → Tasks)
- Takes PRD content, generates structured tasks with: id, title, description, status, dependencies, priority, details, testStrategy
- Optional research mode: researches latest tech/best practices before task breakdown
- Optional codebase analysis: explores existing code via Glob/Grep/Read before generating
- Tasks are atomic, single-responsibility, logically ordered with dependency awareness
- Each task includes `details` (implementation guidance) and `testStrategy` (validation approach)
- Respects explicit PRD requirements strictly; fills gaps with judgment

### analyze-complexity (Task Complexity Scoring)
- Scores each task 1-10 on complexity
- Recommends number of subtasks per task
- Generates expansion prompts to guide subtask generation
- Optional codebase analysis for more accurate scoring

### expand-task (Task → Subtasks)
- Breaks tasks into specific, actionable subtasks
- Uses complexity report's expansion prompts when available
- Each subtask: id, title, description, dependencies, details, status, testStrategy
- Optional research mode + codebase analysis

### research (Context-Aware Research)
- Query-based research with project context
- Three detail levels: low (2-4 paragraphs), medium (4-8), high (8+)
- References specific tasks/files when relevant

## Workflow Pattern

1. list → see tasks
2. next → pick highest priority unblocked task
3. show <id> → understand requirements
4. expand <id> → break down if complex
5. implement → write code
6. update-subtask → log progress/findings
7. set-status done → mark complete
8. repeat

## Key Concepts

- **Implementation drift handling**: When reality diverges from plan, update future tasks
- **Tagged task lists**: Isolation for features, experiments, team members
- **Iterative subtask implementation**: Explore → plan → log → implement → log progress → review → done
- **Complexity-driven expansion**: Only expand tasks scoring above threshold
- **testStrategy per task**: Validation approach defined alongside implementation details (not TDD, just "how to verify")
