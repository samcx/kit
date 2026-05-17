---
name: wt-cleanup
description: Find and remove stale git worktrees and stale local branches whose PRs have been merged into the default branch, using wt/worktrunk, git, and gh CLI. Use when asked to clean up worktrees, remove merged worktrees/branches, or run wt cleanup.
---

# Worktree & Branch Cleanup

Find and remove stale git worktrees and stale local branches using `wt` (worktrunk), `git`, and the `gh` CLI.

`wt remove` deletes the worktree directory and (by default) its branch, but only when git can see the branch is an ancestor of `main`. Squash-merged branches look unmerged to git, and orphan branches whose worktrees were removed earlier never get cleaned up at all. This skill handles both gaps.

## Part 1: Stale worktrees

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

4. If no worktrees qualify, report `No stale worktrees to clean up` and continue to Part 2.

5. Otherwise, show the candidates to the user with branch name, age, and last commit message. Ask which to remove, and offer `All` as an option.

6. Remove the selected worktrees. `wt remove` deletes the branch automatically when git sees it as merged (ancestor of `main`). For squash-merged branches the tip is *not* an ancestor of `main`, so pass `-D` to force-delete the branch too:

```sh
wt remove -f -y -D <branch> [<branch>...]
```

- `-f` — force worktree removal (handles untracked build artifacts)
- `-y` — skip prompts after user has confirmed
- `-D` — force-delete the branch even if it looks unmerged to git (needed for squash merges; safe here because we already verified via `gh` that the PR is merged)

7. Report which worktrees and branches were removed. If any removal fails, show the error.

## Part 2: Stale local branches

After cleaning up worktrees (or if there were none), check for **orphan branches** — branches with no worktree that `wt` won't touch.

1. List local branches, excluding `main` and the current branch:

```sh
git branch --format='%(refname:short)' | grep -v -E '^(main)$'
```

2. A branch is stale if **either** check is true:

- It has a **merged PR** on GitHub (catches squash-merged branches, whose tip SHA is *not* an ancestor of `main`):

  ```sh
  gh pr list --state merged --head <branch> --json number --jq 'length'
  ```

  `> 0` means the branch's PR has been merged.

- It is already an ancestor of `main` (direct or fast-forward merge, no PR needed):

  ```sh
  git branch --merged main --format='%(refname:short)'
  ```

  Any branch in this list is reachable from `main` and can be safely removed.

Run both — squash merges are common and won't show up in `git branch --merged`, while ancestor-merged branches sometimes have no PR at all.

3. If no branches qualify, report `No stale branches to clean up` and stop.

4. Show the stale branches to the user with branch name, short SHA, and last commit message. Note for each whether it matched on merged-PR or ancestor-of-main, so they can decide with full context. Ask which to remove, and offer `All` as an option.

5. Remove the selected branches:

```sh
git branch -D <branch> [<branch>...]
```

Use `-D` (force) because squash-merged branches look unmerged to plain `git branch -d` even though their PR has shipped.

6. Report which branches were removed. If any removal fails (e.g. the branch is currently checked out in another worktree), show the error.
