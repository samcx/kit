---
description: Remove worktrees whose branches have been merged into the default branch
---

Find and remove stale git worktrees using `wt` (worktrunk).

## Agent Workflow (Required)

### Step 1: List worktrees

Run:

```sh
wt list --format=json
```

### Step 2: Filter to removable worktrees

From the JSON output, select entries where ALL of these are true:

- `main_state` is `"integrated"` or `"empty"`
- `is_current` is `false`
- `is_main` is `false`
- `kind` is `"worktree"`

If none match, report "No stale worktrees to clean up" and stop.

### Step 3: Confirm with user

List the candidates (branch name, age, last commit message) and ask the user which to remove. Offer "All" as an option.

### Step 4: Remove

Run `wt remove -f -y <branch> [<branch>...]` with the selected branches. The `-f` flag handles untracked files (build artifacts), and `-y` skips interactive prompts.

### Step 5: Report outcome

Report which worktrees were removed. If any removals failed, show the error.
