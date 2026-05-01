---
name: wt-cleanup
description: Find and remove stale git worktrees whose branches have been merged into the default branch using wt/worktrunk. Use when asked to clean up worktrees, remove merged worktrees, or run wt cleanup.
---

# Worktree Cleanup

Find and remove stale git worktrees using `wt` (worktrunk).

## Workflow

1. Fetch the latest remote refs **and fast-forward the local main branch** so `wt list` correctly detects merged branches. `wt` compares against the local `main` ref, not `origin/main`, so a stale local main causes merged branches to appear as `"diverged"` instead of `"integrated"`.

```sh
git fetch origin main
```

Then fast-forward local main. The approach depends on where you are:

- **If currently on `main`**: `git merge --ff-only origin/main`
- **If in a different worktree**: run the merge from the main worktree path (first entry in `git worktree list`):

```sh
main_path=$(git worktree list --porcelain | awk '/^worktree /{print $2; exit}')
git -C "$main_path" merge --ff-only origin/main
```

2. List worktrees:

```sh
wt list --format=json
```

3. From the JSON output, select removable worktrees where all of these are true:

- `main_state` is `"integrated"` or `"empty"`
- `is_current` is `false`
- `is_main` is `false`
- `kind` is `"worktree"`

4. If no worktrees match, report `No stale worktrees to clean up` and stop.

5. Show the candidates to the user with branch name, age, and last commit message. Ask which to remove, and offer `All` as an option.

6. Remove the selected worktrees:

```sh
wt remove -f -y <branch> [<branch>...]
```

Use `-f` because stale worktrees often contain untracked build artifacts. Use `-y` to avoid interactive prompts after the user has already confirmed the selection.

7. Report which worktrees were removed. If any removal fails, show the error.
