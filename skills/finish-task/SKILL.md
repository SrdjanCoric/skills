---
name: finish-task
description: Finish an implemented and reviewed task — README audit, final behavior proof, completion record, PR creation after approval, and cleanup of the task's retained review files after CI passes. Use manually after the task-review and review-fix-worker loop is clean. Lightweight orchestration suited to a small fast model.
---

# Finish Task

Take one implemented task whose manual review loop is complete and carry it to a CI-green PR. This
skill is lightweight orchestration: it updates the README, proves final behavior, records
completion, hands off to `create-pr` after explicit user approval, and removes the task's retained
review files after the PR is CI-green. It does not implement, review, or remediate.

## Model routing

This skill is mechanical orchestration over already-verified work. Run it on the smallest capable
fast model (router: `finish-task` maps to the luna-max route). Escalate to the user rather than
muscling through ambiguity.

## Preconditions

1. Resolve the task from a supplied path, or find the single `[~]` task in the master plan. With
   zero or multiple `[~]` tasks and no supplied path, stop and report the state.
2. Confirm the current branch matches the task's `**Branch**` value; otherwise stop.
3. Normalize the selected task to its repository-relative path. Inspect every `reviews/*.md` file
   at the repository root and select those whose `**Task:**` matches that normalized path. If any
   matching file contains a finding with `**Status:** open`,
   stop and tell the user to run `/skill:review-fix-worker` (or explicitly skip or accept the
   remaining findings) first. Absence of a matching review file for a `code` or `mixed` task means
   `task-review` never ran; stop and tell the user to run `/skill:task-review` first. Retain the
   matching file list and each review outcome for completion recording, PR evidence, and cleanup.

## Workflow

### Mandatory execution checklist

Create this checklist in working context when the skill starts and keep it current. Check an item
only after completing the corresponding workflow section with evidence. If work stops, report the
first unchecked item and its blocker.

- [ ] Preconditions confirmed: task, branch, review loop closed
- [ ] README inspected; `write-well` audit completed only when README prose changed, or no-impact reason recorded
- [ ] Highest-level automated and required manual proof passed
- [ ] Task file records implementation, decisions, documentation, review, and proof
- [ ] User approved PR creation and the current head is CI-green
- [ ] All retained review files for this task removed; other tasks' files untouched

### 1. Update the README

Inspect the README now that the implementation has reached its reviewed
state. When the task changed current application behavior, setup, configuration, or usage, invoke
`write-well` only for the README update and audit only the affected prose. Complete the skill's full
audit loop; loading `write-well` alone does not complete this step. Describe the current application,
not a history of what changed. Record the affected sections and audit pass count in the task file.

Leave the README unchanged when the task has no documentation impact and record that conclusion in
the task file.

### 2. Prove the final behavior

Run the final validation for the task's recorded tier, once:
document/plan consistency for `documentation`, affected dependency or configuration proof for
`focused`, and the repository's canonical check for `canonical`. A documentation or focused tier
must not be promoted to the application's canonical suite without an actual diff reclassification
or explicit acceptance criterion. Run the highest-level automated proof, plus any
focused check invalidated by a later change. Prefer browser-level proof for
user-facing behavior and executable scripts or disposable local environments for CLI, API,
database, and provider workflows.

Keep `[verify]` items from the task for this step.

When automated verification is impossible, give the user exact steps, the expected result, and
the signal that indicates failure. Explain why the agent cannot perform the check, then wait for
the user to confirm it passed. Manual verification blocks completion.

### 3. Record completion

Check off the task's implementation work, human checkpoints, and acceptance criteria. Record what
was built, decisions made, relevant file paths, README disposition, review outcome (findings fixed,
skipped, or accepted as security risks, with reasons), automated proof, and manual verification.
Leave the master-plan pointer at `[~]` until a PR exists and its CI is green.

### 4. Open the PR after approval

Summarize the completed task and ask the user to approve opening the PR. Do not invoke `create-pr`
without explicit approval.

After approval, invoke `create-pr` with the task path, the review outcome (fixed findings, skipped
minors with reasons, accepted security risks with user reasons), verification
proof, and accepted security risks. `create-pr` opens or updates the PR and
waits for CI. After a green run, it changes the managed task pointer from `[~]` to `[>]` and pushes
the marker commit. It waits for CI on the new head and does not merge.

Treat PR creation as complete only when `create-pr` returns a CI-green current head and confirms the
task pointer is `[>]`. Leave the task file under `plans/tasks/`. If PR creation or CI does not
succeed, the pointer must remain `[~]` and every review file must remain in place. The user invokes
`sync-main` separately to merge and close the task.

### 5. Remove the task's review files

Only after `create-pr` confirms the current PR head is CI-green and the managed task pointer is
`[>]`, delete every retained `reviews/*.md` file whose `**Task:**` exactly matches the selected
task's normalized repository-relative path. These files are ignored workflow state and their outcomes are already recorded in the task
file and PR evidence. Never delete a review file for another task or branch. Leave `reviews/`, its
`.gitignore` entry, and unrelated workflow files intact.

Confirm every matching review file is gone before reporting completion.

## Rules

- Finish one task per invocation.
- Never work on `main`.
- Do not run while any review finding for the selected task is still open.
- Keep all review files until the selected task's current PR head is CI-green; then remove every
  review file matching that task and no others.
- Verify completion against the code and observable behavior, not the implementation log.
- Stop for impossible-to-automate verification and PR approval.
- Keep `[>]` distinct from `[x]`; only `sync-main` closes a merged task.
