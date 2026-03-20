# GitHub Output — Sub-Agent Instructions

Blueprint spawns a sub-agent to create the GitHub artifacts. This file contains the sub-agent's instructions.

## PR on the Target Repo

```
PR: "plan: [feature-name]"
Branch: plan/[feature-slug]
Labels: plan:draft
```

## Files in PR

```
specs/FEAT-[feature-name].md    ← The PRD
plans/PLAN-[feature-name].md    ← High-level plan summary (what/why, not how)
```

## Plan Issue

```
Title: "[PLAN] Feature Name"
Labels: plan:draft
Body:
  - Executive summary
  - Link to PR
  - Validation score + grade
  - Reviewer verdicts summary
  - Task count estimate
```

## Sub-Agent Behavior

1. Create the branch from the repo's default branch.
2. Write the PRD to `specs/FEAT-[feature-name].md`.
3. Write the plan summary to `plans/PLAN-[feature-name].md`.
4. Commit, push, and create the PR with `gh pr create`.
5. Create the plan issue with `gh issue create`.
6. Return the PR URL and issue URL to Blueprint as the final message.
