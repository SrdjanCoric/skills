---
name: implement-next-task
description: Implement the next eligible local task in the current checkout, use talk-it-through when the task or an unexpected obstacle leaves the approach uncertain, verify the behavior, run autonomous task review, update the README, and open a CI-green PR after user approval. Use with a task ordinal/path or when asked to build the next task.
---

# Implement Next Task

Implement one eligible task in full on its feature branch. Use one implementation workflow at a
time in the current checkout. The master plan controls order and status; the selected task file is
the contract for what to build.

Without explicit user approval, modify only repository-scoped files and isolated local/test
databases; never run destructive shell commands, access production systems or data, read `.env`
files, or delete or move anything outside the repository root.

## Local task model

Find the master plan through the most specific `AGENTS.md` `Active plan` entry, falling back to
`CLAUDE.md`. Task files live under `plans/tasks/`.

- `[ ]` means ready and unclaimed.
- `[~]` means in progress.
- `[>]` means complete with a CI-green PR open and awaiting merge.
- `[x]` means merged into `main`; `sync-main` moves the task to `plans/tasks/done/`.

A task is eligible only when its pointer is `[ ]` and every ordinal in its `(after ...)` suffix is
`[x]`. A dependency at `[ ]`, `[~]`, or `[>]` blocks it because the required code is not on `main`.

Accept an optional task ordinal or task-file path. If supplied, use that task only when it is
eligible. Otherwise choose the first eligible pointer. If none is eligible, report the blocking
tasks. When a `[>]` task appears to be the blocker, suggest `sync-main` because its PR may already
be ready or merged.

## Workflow

### Mandatory execution checklist

Create this checklist in working context when the skill starts and keep it current. Check an item
only after completing the corresponding workflow section with evidence. If work stops, report the
first unchecked item and its blocker. Do not copy this process checklist into committed files;
record durable decisions and proof in the task file.

- [ ] Task selected, claimed, and read in full
- [ ] Current code and test reality inspected
- [ ] Outcome, branch, and verification method confirmed
- [ ] Decisions and unexpected obstacles resolved through `talk-it-through`, or not applicable
- [ ] Repository guidelines loaded with expected proof
- [ ] `tdd` loaded and behavior completed through observed red-green-refactor cycles
- [ ] Implementation work completed within scope
- [ ] Focused and full validation passed
- [ ] `task-review` completed without unresolved findings
- [ ] README inspected after review; `write-well` audit completed, or no-impact reason recorded
- [ ] Highest-level automated and required manual proof passed
- [ ] Task file records implementation, decisions, documentation, review, and proof
- [ ] User approved PR creation and the current head is CI-green

### 1. Select and claim the task

Read the master plan, choose the task, and immediately change its pointer from `[ ]` to `[~]`.
Read the architectural decisions and the selected task file in full. Read referenced decision
documents only when the task depends on them. Do not load the source PRD or neighboring tasks to
expand the task's scope.

If the task cannot be claimed or the current checkout already has another active implementation
workflow, stop and report the state.

### 2. Verify current reality

Inspect the code and tests before planning edits. Use one `Explore` or `general-purpose` subagent
for the broad reconnaissance pass when available, and ask it for a concise digest containing:

- differences between the task's assumptions and the current code;
- relevant files and code ranges;
- existing test seams and conventions;
- the minimum excerpts needed for the first failing test and implementation.

Trust the code over stale task notes. Read files directly just before editing them.

### 3. Confirm the selected task

State the task ordinal, title, independently verifiable outcome, branch, and verification method.

### 4. Create or check out the branch

Use the task's `**Branch**` value. Check out an existing local or remote branch when it matches the
task. Otherwise update `main` and create the branch from it. Never implement on `main`.

### 5. Resolve uncertainty and human checkpoints

If the task does not provide enough information to choose a safe implementation approach, stop
before writing affected code and invoke `talk-it-through`.

If implementation encounters an unexpected obstacle that the task does not address, stop and
invoke `talk-it-through` before changing scope, behavior, architecture, dependencies, or safety
assumptions. Explain the uncertainty or obstacle, discuss one decision at a time, recommend an
approach, wait for shared understanding, and record the decision in the task file. Do not invent
requirements, reinterpret the task, or expand its scope to bypass an obstacle. Continue routine
implementation choices autonomously when the task and repository conventions provide enough
direction.

Resolve declared `[decision]` items through `talk-it-through` before writing code they affect.
Require explicit approval before `[confirm-db]` work on real, shared, destructive, persistent, or
ambiguous data and before `[confirm-security]` work that changes a trust boundary. Isolated local
or test-database work may proceed within the task's scope. Keep `[verify]` items for the final proof
step.

### 6. Implement the task

Invoke `software-repository-guidelines` in `implement` mode with the task, repository state,
declared references, and affected capabilities. Record the references loaded, applicable
requirements, and expected proof.

Load `tdd` and follow its red-green-refactor loop for behavior changes. Loading `tdd` alone does
not complete this step: observe RED and GREEN for each behavior slice and retain the command evidence
in working context. Implement every item under `Implementation work`, follow the architectural
decisions, and stay within the selected task. Automate any verification that can be automated.

### 7. Verify the implementation

Run focused tests and checks while working, then run the relevant full validation for the affected
area. Verify every applicable repository-guideline requirement with repository, command, CI, or
provider evidence. Fix current-task gaps and leave unrelated repository improvements alone.

### 8. Run task review

Invoke `task-review` with:

- `base=main`;
- `spec=` the verbatim task file and referenced decision documents;
- `repository-guidelines=` the implement-mode result and recorded proof;
- `task=` the task-file path.

`task-review` reviews the implementation and tests, fixes every supported non-security finding
attributable to this task, and reruns its panel for at most two remediation passes. It writes no
review document. If it does not converge, stop and follow its explanation.

For every security finding, wait while `task-review` explains the risk and consequences through
`write-well`. Continue only after the user approves a fix or accepts the risk. Pass accepted risks
and their reasons to `create-pr`.

### 9. Update the README

Inspect the README after task-review remediation, when the implementation has reached its reviewed
state. When the task changed current application behavior, setup, configuration, or usage, invoke
`write-well` and update only the affected sections. Complete the skill's full audit loop; loading
`write-well` alone does not complete this step. Describe the current application, not a history of
what changed. Record the affected sections and audit pass count in the task file.

Leave the README unchanged when the task has no documentation impact and record that conclusion in
the task file.

### 10. Prove the final behavior

After review remediation, run the highest-level automated proof available. Prefer browser-level
proof for user-facing behavior and executable scripts or disposable local environments for CLI,
API, database, and provider workflows.

When automated verification is impossible, give the user exact steps, the expected result, and
the signal that indicates failure. Explain why the agent cannot perform the check, then wait for
the user to confirm it passed. Manual verification blocks completion.

### 11. Record completion

Check off the task's implementation work, human checkpoints, and acceptance criteria. Record what
was built, decisions made, relevant file paths, guideline proof, README disposition, automated
proof, manual verification, and accepted security risks. Leave the master-plan pointer at `[~]`
until a PR exists and its CI is green.

### 12. Open the PR after approval

Summarize the completed task and ask the user to approve opening the PR. Do not invoke `create-pr`
without explicit approval.

After approval, invoke `create-pr` with the task path, task-review result, repository-guideline
evidence, verification proof, and accepted security risks. `create-pr` opens or updates the PR and
waits for CI. After a green run, it changes the managed task pointer from `[~]` to `[>]` and pushes
the marker commit. It waits for CI on the new head and does not merge.

Treat PR creation as complete only when `create-pr` returns a CI-green current head and confirms the
task pointer is `[>]`. Leave the task file under `plans/tasks/`. If PR creation or CI does not
succeed, the pointer must remain `[~]`. The user invokes `sync-main` separately to merge and close
the task.

## Rules

- Implement one task per invocation.
- Do not run multiple implementation workflows in the same checkout.
- Never work on `main`.
- Do not start a dependency-blocked task.
- Stay within the selected task's independently verifiable behavior.
- Keep the mandatory execution checklist current and report the first unchecked item when blocked.
- Use `talk-it-through` for unresolved approach decisions and unexpected out-of-task obstacles.
- Run task review before the one final README update.
- Stop for declared decisions, unresolved implementation uncertainty, unexpected out-of-task
  obstacles, impossible-to-automate verification, destructive database actions, security
  boundaries, review security findings, and PR approval.
- Verify completion against the code and observable behavior, not the implementation log.
- Keep `[>]` distinct from `[x]`; only `sync-main` closes a merged task.
