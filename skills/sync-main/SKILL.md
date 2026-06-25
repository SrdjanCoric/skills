---
name: sync-main
description: Checkout main, pull the latest changes, remove the current worktree if running from one, clean up merged local branches, and close out merged tasks in the master plan ([>]→[x]). Use when the user wants to sync with main, get back on main, or mentions "sync main".
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

Finally, close out the merged tasks in the master plan (the project named in `CLAUDE.md` "Active plan"; its `plans/` lives in the main working tree you are now in). Build the set of branches to close out from the merged branches confirmed above **plus**, if you came through the worktree path, the branch you captured in worktree-path step 1 — that one was already deleted in step 6, so it will NOT appear in `git branch --merged`, and you run `sync-main` from a worktree only after its PR has merged, so it belongs in this set. For each such branch:

1. Find the task whose `**Branch**` field matches that branch — its pointer in the plan's `## Tasks` list should be `[>]` (done, awaiting merge).
2. Flip that pointer `[>]→[x]` (merged), move the task file from `plans/tasks/` to `plans/tasks/done/`, and repoint the link to `tasks/done/`.
3. If the pointer is already `[x]`, or no matching task exists, skip it silently. If it is still `[ ]`/`[~]` (not `[>]`), the task wasn't completed through the normal flow — leave it and tell the user rather than guessing.

This is the only place `[>]→[x]` happens: a task is "merged" exactly when its branch has landed on `main`, which is what unblocks any dependent `(after …)` task.
