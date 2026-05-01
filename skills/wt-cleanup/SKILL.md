---
name: wt-cleanup
description: Find and remove stale git worktrees whose branches have been merged into the default branch using wt/worktrunk. Use when asked to clean up worktrees, remove merged worktrees, or run wt cleanup.
---

# Worktree Cleanup

Find and remove stale git worktrees using `wt` (worktrunk) and `gh` CLI.

## Workflow

1. List worktrees:

```sh
wt list --format=json
```

2. From the JSON output, collect candidate worktrees where all of these are true:

- `is_current` is `false`
- `is_main` is `false`
- `kind` is `"worktree"`

3. For each candidate, check if its branch has a **merged PR** on GitHub:

```sh
gh pr list --state merged --head <branch> --json number --jq 'length'
```

A result `> 0` means the branch was merged. Keep only worktrees whose branches have merged PRs.

4. If no worktrees qualify, report `No stale worktrees to clean up` and stop.

5. Show the candidates to the user with branch name, age, and last commit message. Ask which to remove, and offer `All` as an option.

6. Remove the selected worktrees:

```sh
wt remove -f -y <branch> [<branch>...]
```

Use `-f` because stale worktrees often contain untracked build artifacts. Use `-y` to avoid interactive prompts after the user has already confirmed the selection.

7. Report which worktrees were removed. If any removal fails, show the error.
