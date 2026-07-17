---
name: create-pr
description: Create or update a pull request from the current branch, wait for CI, and invoke diagnose for in-scope CI failures. Use when the user wants to open or prepare a PR; stop after the current head is CI-green and never merge or invoke sync-main.
---

# Create Pull Request

Commit the current unit of work, push its branch, open or update its PR, and wait for CI. Stop when
the current PR head is ready to merge. Do not merge the PR or invoke `sync-main`.

## Inputs

Accept an optional feature name and optional evidence from `implement-next-task`:

- task-review result;
- repository-guideline proof;
- automated and manual verification proof;
- accepted security risks and the user's reasons.

## Process

### 1. Gather current state

Run:

```sh
git branch --show-current
git status --short
git diff --cached --stat
git diff --stat
git log main..HEAD --oneline || git log origin/main..HEAD --oneline
git diff main...HEAD || git diff origin/main...HEAD
```

Confirm the current branch is not `main`. Identify the observable result, dependencies, and
material risks in the diff.

When the caller supplies managed review or repository-guideline evidence, confirm there is no
unresolved supported finding, security decision, or current-task proof gap. Return to the caller
when supplied evidence is incomplete. Standalone PR creation does not invent missing managed
evidence.

### 2. Stage and commit

Stage only changes that belong to the PR. Include related README changes. Leave unrelated changes
unstaged. Never stage secrets, credentials, local environment files, or unrelated plan files.

Commit when needed using repository conventions, falling back to `type(scope): description`.
Never mention an AI assistant in a commit message.

### 3. Push the branch

Push all commits to `origin` and create upstream tracking when needed.

### 4. Write the PR

Invoke `write-well` before writing the title, description, or comments. Follow repository title
conventions, falling back to `type(scope): brief description`.

Use this body:

```markdown
## Summary
<What the current branch delivers and why.>

## Changes
- <Behavior or capability now present>

## Verification
- <Command or manual proof and result>

## Risk
<Applicable security, data, compatibility, deployment, and rollback risks, or "Low" with a reason.>
```

Describe the repository's resulting behavior, not the agent's process. Include exact verification
commands and results. Include every accepted security risk and the user's reason in `Risk`. Do not
mention an AI assistant.

### 5. Open or update the PR

Use `gh pr create` or the repository's documented equivalent. Add configured labels and reviewers.
If the branch already has an open PR, update and reuse it.

Capture the PR URL, number, head branch, base branch, and head SHA. Use `main` as the base unless
the user explicitly chose another branch.

### 6. Wait for CI

Wait for all checks on the current head to reach a terminal state. Use
`gh pr checks <pr> --watch --interval 10` or the repository's documented equivalent. Then confirm:

- the head SHA has not changed;
- no check is pending or queued;
- required checks passed;
- no failure or unexpected cancellation remains.

If the head changes, wait for the new head. If the repository has no CI and no required checks,
record that state and continue.

### 7. Diagnose CI failures

When a check fails or is unexpectedly cancelled:

1. Capture the check names, URLs, annotations, and failed logs.
2. Invoke `diagnose` with the PR, head SHA, failing command, and evidence.
3. If diagnosis produces an in-scope fix, verify it, commit it, push it, and return to the CI wait.
4. If credentials, unavailable infrastructure, policy, or another external condition blocks the
   PR, report the blocker and stop.

Never bypass branch protection, weaken tests, or merge while CI is failing, pending, or stale.

### 8. Return the ready PR

After CI passes for the current head, return:

- PR URL and number;
- base branch, head branch, and head SHA;
- CI result;
- verification evidence;
- accepted security risks included in the PR.

Stop without merging. The user invokes `sync-main` separately when ready.
