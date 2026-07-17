---
name: sync-main
description: Merge a verified pull request remotely, synchronize local main, clean verified merged branches, and move the matching local task to done. Use when the user explicitly asks to merge a ready PR, sync main, return to main, or close a merged task.
---

# Sync Main

Treat explicit invocation as authorization to merge the verified PR. Do not ask for another routine
merge confirmation. Stop when local state is unsafe, the PR is ambiguous, CI is not green, or
repository merge requirements are unmet.

## 1. Protect local state

Run `git status --porcelain`. Before merging or switching branches, determine whether uncommitted
changes would be overwritten or stranded. If so, ask the user whether to commit, stash, or abort.
Do not merge remotely until local state is safe for the full workflow.

Record the starting branch and any supplied PR identity, expected head branch, and expected head
SHA.

## 2. Resolve and verify the PR

Use a PR URL or number supplied by the user or caller. If none is supplied and the starting branch
is not `main`, resolve the open PR for that branch. Do not guess between multiple PRs. If no PR can
be resolved, skip remote merge and perform only the requested local synchronization and safe
cleanup.

Before merging, verify that:

- the PR is open and not a draft;
- its base is `main`, unless the user explicitly chose another base;
- its head branch and SHA match supplied expectations;
- every required CI check passed for the current head;
- no check is pending, queued, failed, or unexpectedly cancelled;
- review and branch-protection requirements are satisfied;
- the hosting provider reports the PR as mergeable.

If CI failed, report the evidence and suggest `create-pr` or `diagnose`. If CI became pending, wait
for it. Never bypass branch protection, force-merge, or merge stale code.

## 3. Merge remotely

Use `gh pr merge <pr> --delete-branch` or the repository's documented equivalent and merge
strategy. If no strategy is documented, inspect repository settings and recent merged PRs. Ask only
when more than one strategy remains plausible and the choice would change project history.

Capture the final PR state and merge commit. Confirm through the hosting provider that the PR is
merged. Treat a failed or queued merge as incomplete and stop.

## 4. Synchronize local main

Run:

```sh
git checkout main
git fetch --prune
git pull --ff-only
```

If local `main` diverged, diagnose the state. Never reset or discard commits automatically. Ask the
user when safe recovery is ambiguous.

When a PR was merged, verify that the merge commit is reachable from local `main`. For squash or
rebase merges, use the verified remote PR state and captured merge commit as evidence.

## 5. Clean merged branches

Run `git branch --merged main`. Exclude `main` and the current branch. Delete safely merged local
branches with `git branch -d` and report them.

For the exact verified PR head after a squash or rebase merge, a matching remote head SHA and
merged PR state are sufficient evidence. If `git branch -d` refuses only because ancestry changed,
delete that exact branch with `git branch -D`. Do not use this exception for any other branch.

Inspect branches whose upstream is `[gone]`. If one is not otherwise verified as merged, report it
and ask before force deletion. Remote branch deletion alone is not merge proof.

## 6. Close merged local tasks

Find the master plan through the most specific `AGENTS.md` `Active plan` entry, falling back to
`CLAUDE.md`. For each verified merged branch in the closeout set:

1. Find the active task whose `**Branch**` matches the merged branch. Its pointer should be `[>]`.
2. Change `[>]` to `[x]`.
3. Move the task file from `plans/tasks/` to `plans/tasks/done/`.
4. Update the master-plan pointer to the done path.

If a pointer is already `[x]` or no task matches, skip it. If it is `[ ]` or `[~]`, leave it
unchanged and report the lifecycle mismatch.

This is the only workflow step that changes `[>]` to `[x]`. A dependent task becomes eligible only
after its prerequisite is verified merged into local `main`.

If closeout edits affect tracked files on `main`, follow repository policy. Do not push them
directly unless that policy permits it.

## Final result

Report:

- PR and merge commit;
- updated local `main` state;
- local and remote branches removed;
- tasks changed to `[x]` and moved to `plans/tasks/done/`;
- any safety or lifecycle mismatch left unresolved.
