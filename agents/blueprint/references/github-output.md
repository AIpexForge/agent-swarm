# GitHub Output — Finalization Instructions

By Phase 7, the draft PR already exists from the User Review Gate (Phase 4). This phase finalizes it.

## Finalize the Existing PR

1. Push any remaining changes to the existing `plan/[feature-slug]-draft` branch:
   - Updated PRD (post-validation/review fixes) → `specs/FEAT-[feature-name].md`
   - Plan summary → `plans/PLAN-[feature-name].md`

2. Mark the PR as ready:
   ```bash
   gh pr ready
   ```

3. Update PR body with validation score and reviewer verdicts if not already there.

## Create Plan Issue

```bash
gh issue create \
  --repo <repo> \
  --title "[PLAN] Feature Name" \
  --label "plan:draft" \
  --body "<ISSUE_BODY>"
```

**Issue body includes:**
- Executive summary
- Link to PR
- Validation score + grade
- Reviewer verdicts summary
- Task count estimate

## Return Values

Return the PR URL and issue URL to Blueprint as the final message.
