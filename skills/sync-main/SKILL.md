---
name: sync-main
description: Checkout to main branch and pull the latest changes from remote, removing the current worktree first if running from one, then cleaning up merged local branches. Use when the user wants to sync with main, get back on main, or mentions "sync main".
---

Checkout to main branch and pull the latest changes from remote, then clean up local branches that have already been merged. If the command is being run from inside a git worktree, remove that worktree first.

Before switching branches or removing a worktree (either path below), check for uncommitted changes that would be overwritten. If there are any, warn the user and ask whether to stash them first.

First, determine whether the current directory is a linked git worktree (not the main working tree):

- Run `git rev-parse --path-format=absolute --git-dir --git-common-dir` and compare the two absolute paths. If they differ, you are inside a linked worktree. (Use `--path-format=absolute` so the comparison is bulletproof — `--git-common-dir` can otherwise come back relative on some git versions.)

If you ARE inside a linked worktree:
1. Capture the current branch name (`git rev-parse --abbrev-ref HEAD`) and the worktree path (`git rev-parse --show-toplevel`) — you will need both after leaving.
2. Move to the main working tree (the parent directory of the absolute `--git-common-dir`) — you cannot remove a worktree while the shell is still inside it.
3. Remove the worktree with `git worktree remove <path>`. If it has uncommitted changes that block removal, warn the user and ask before using `git worktree remove --force <path>`.
4. `git checkout main` - switch to the main branch.
5. `git pull` - pull the latest changes from remote.
6. Delete the branch the worktree was on with `git branch -d <branch>`. If git refuses because the branch isn't fully merged, warn the user and ask before using `git branch -D <branch>`.

If you are NOT inside a worktree (normal repo):
1. `git checkout main` - switch to the main branch.
2. `git pull` - pull the latest changes from remote.

After syncing main (in BOTH cases), clean up merged local branches:
1. `git fetch --prune` - drop remote-tracking refs for branches deleted on the remote.
2. `git branch --merged main` - list local branches already merged into main. Exclude `main` itself and the current branch.
3. Delete each merged branch with `git branch -d <branch>`, and tell the user which branches were deleted (e.g. "Deleted merged branches: X, Y").
4. Squash-merged or rebase-merged branches will NOT show up as merged, because their commits don't appear verbatim on main. To catch these, run `git branch -vv` (double `v` — only `-vv` prints the upstream-tracking column) and look for branches whose upstream is marked `[gone]`. A gone upstream means the remote branch was deleted — usually after a merge, but not guaranteed — so list these to the user and ask before deleting with `git branch -D <branch>`. Never force-delete without confirmation.
