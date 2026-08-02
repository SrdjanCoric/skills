---
name: implement-next-task
description: Select, claim, and set up the next eligible local task, then implement it through tdd-worker with TDD and focused validation while minimizing parent-context growth and avoidable wall time. Stops before review; the user then runs task-review, review-fix-worker, and finish-task manually.
---

# Implement Next Task

Select and implement one eligible task in full on its feature branch, then stop and hand off to the
manual review loop. Use one implementation workflow at a time in the current checkout. The master
plan controls order and status; the selected task file is the contract for what to build.

This skill covers selection, setup, and implementation. It does not review, update the README,
record final completion, or open a PR. When it ends, tell the user to run:

1. `/skill:task-review` — writes a findings file under `reviews/`;
2. `/skill:review-fix-worker` — fixes the findings one by one;
3. repeat 1–2 until the review is clean or only accepted/skipped findings remain;
4. `/skill:finish-task` — README, final proof, completion record, and PR.

Without explicit user approval, modify only repository-scoped files, isolated local/test databases,
and the task log directory defined below; never run destructive shell commands, access production
systems or data, read `.env` files, or delete or move anything else outside the repository root.

## Efficient evidence handling

Keep the main context for decisions, behavioral TDD evidence, targeted source inspection, and final
integration proof. Delegate broad, read-only interpretation to isolated subagents when available and
require concise structured results. Do not delegate primary RED-GREEN decisions, concurrent edits,
or stateful checks that share a database, port, simulator, fixture, generated output, or external
service.

Use the smallest evidence that proves a result. After branch checkout, broad commands and full test
output belong in the task-scoped log directory; return the exit status, duration, concise result,
relevant failure excerpt, and log path. Never create another workflow log location. Before checkout,
reconnaissance returns bounded results directly and does not retain full command output. For
searches, list matching files and counts first, then inspect matching lines only in selected files.
Batch independent tool calls in the same turn and read exact ranges instead of whole files when
possible.

Use stable names such as `unit.log`, `typecheck.log`, or `review-security.log` and overwrite a
superseded passing log instead of accumulating timestamped copies. Enforce the 5 MiB cap while each
command runs rather than writing an unbounded file and truncating it afterward. Retain the latest
output when truncation is necessary, and keep the task directory under 50 MiB by removing
superseded passing logs first. Never write secrets, credentials, environment contents, source files,
or database exports to these logs. Preserve current failure evidence until the failure is resolved.

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
- [ ] Outcome, branch, verification method, and task log directory confirmed
- [ ] Decisions and unexpected obstacles resolved through `talk-it-through`, or not applicable
- [ ] `tdd-worker` completed for coding behavior, or correctly skipped for non-coding work
- [ ] Implementation work completed within scope
- [ ] Focused pre-review validation passed
- [ ] Manual review-loop handoff (task-review → review-fix-worker → finish-task) reported to the user

### 1. Select and claim the task

Read the master plan, choose the task, and immediately change its pointer from `[ ]` to `[~]`.
Read the architectural decisions and the selected task file in full. Read referenced decision
documents only when the task depends on them. Do not load the source PRD or neighboring tasks to
expand the task's scope.

Before broad repository exploration, inspect the selected task and base branch, then classify the
expected work as one of:

- `plan-only`: task and plan lifecycle content only;
- `documentation-only`: human-facing prose, diagrams, or examples with no executable surface;
- `dependencies`: manifests, lockfiles, patches, or dependency policy only;
- `configuration`: declarative tooling, workflow, or environment configuration without production code;
- `code`: production or test code;
- `mixed`: more than one material class.

Derive `validationTier` and `tddApplicable`, then check that both are consistent with the class.
Fail closed to `mixed` and `canonical` when ambiguity could hide executable behavior. Record the
result immediately. Classification never weakens security review or an explicit task acceptance
criterion.

If the task cannot be claimed or the current checkout already has another active implementation
workflow, stop and report the state.

### 2. Verify current reality

Inspect the code and tests before planning edits. Perform one broad reconnaissance pass, using an isolated read-only
subagent when available. Return a concise digest rather than raw command output,
containing:

- differences between the task's assumptions and the current code;
- relevant files and code ranges;
- existing test seams and conventions;
- the minimum excerpts needed for the first failing test and implementation.

Trust the code over stale task notes. Read directly only the exact source and test ranges needed for
editing or verification. Keep exact, bounded searches in the main context; delegate broad searches
requiring interpretation, such as migration, dead-code, caller, test-surface, or documentation
audits, when isolated subagents are available. Require every delegated audit to return `Status`,
`Actionable findings`, and `Expected or ignored matches`, with paths and line numbers only.

### 3. Confirm the selected task

State the task ordinal, title, independently verifiable outcome, branch, and verification method.

### 4. Create or check out the branch

Use the task's `**Branch**` value. Check out an existing local or remote branch when it matches the
task. Otherwise update `main` and create the branch from it. Never implement on `main`.

After checkout, create or reuse the deterministic task log directory. Refuse to use any existing
symbolic link in its path:

```sh
LOG_ROOT=/tmp/agent-workflows
REPO_ROOT="$(git rev-parse --show-toplevel)"
BRANCH="$(git branch --show-current)"
REPO_KEY="$(printf '%s' "$REPO_ROOT" | git hash-object --stdin | cut -c1-12)"
BRANCH_KEY="$(printf '%s' "$BRANCH" | git hash-object --stdin | cut -c1-12)"
REPO_LOG_DIR="$LOG_ROOT/$REPO_KEY"
TASK_LOG_DIR="$REPO_LOG_DIR/$BRANCH_KEY"
for path in "$LOG_ROOT" "$REPO_LOG_DIR" "$TASK_LOG_DIR"; do
  if [ -L "$path" ]; then
    printf 'Refusing symbolic-link log path: %s\n' "$path" >&2
    exit 1
  fi
done
if [ -e "$TASK_LOG_DIR" ]; then
  [ -d "$TASK_LOG_DIR" ] && [ -f "$TASK_LOG_DIR/repo-root" ] && [ -f "$TASK_LOG_DIR/branch-name" ] &&
    [ "$(cat "$TASK_LOG_DIR/repo-root")" = "$REPO_ROOT" ] &&
    [ "$(cat "$TASK_LOG_DIR/branch-name")" = "$BRANCH" ] || {
      printf 'Refusing mismatched task log directory: %s\n' "$TASK_LOG_DIR" >&2
      exit 1
    }
else
  mkdir -p "$TASK_LOG_DIR"
  printf '%s\n' "$REPO_ROOT" > "$TASK_LOG_DIR/repo-root"
  printf '%s\n' "$BRANCH" > "$TASK_LOG_DIR/branch-name"
fi
```

Retain `TASK_LOG_DIR` in working context. Recompute it with the same commands when needed and pass
it explicitly to every subagent or child skill that may produce verbose output. Reuse an existing
directory only when both metadata files exactly match the current repository root and branch;
otherwise stop rather than writing into it. Keep the directory through implementation, the manual
review loop, PR creation, and CI so failures remain diagnosable. `sync-main` removes it only after
successful merge and synchronization.

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
step in `finish-task`.

### 6. Implement through tdd-worker

Invoke `tdd-worker` with the task-file path, the recorded classification (`change-class`,
`validation-tier`, `tddApplicable`), the reconnaissance digest, and `TASK_LOG_DIR`. `tdd-worker`
loads `tdd` when applicable, runs the red-green-refactor loop for every item under the task's
`Implementation work`, and runs the focused pre-review validation for the recorded tier.

For plan-only, documentation-only, dependency-only, or declarative configuration-only tasks,
`tdd-worker` skips `tdd` and verifies changes with targeted syntax, consistency, security,
configuration, or provider checks instead.

When `tdd-worker` returns, confirm its reported evidence: every work item implemented, RED and
GREEN observed for each coherent behavior cycle, and the tier validation passing. Do not re-run its
passing validation.

### 7. Hand off to the manual review loop

When implementation and focused validation are complete, commit the work on the feature branch and
stop. Report:

- the task ordinal, title, and branch;
- what was built and the proof gathered;
- deferred decisions or obstacles, if any;
- the exact next commands: `/skill:task-review`, then `/skill:review-fix-worker`, repeating until
  clean, then `/skill:finish-task`.

Do not invoke `task-review`, update the README, run final proof, record completion, or open a PR in
this skill. Those belong to the manual review loop and `finish-task`.

## Rules

- Implement one task per invocation.
- Do not run multiple implementation workflows in the same checkout.
- Never work on `main`.
- Do not start a dependency-blocked task.
- Stay within the selected task's independently verifiable behavior.
- Keep the mandatory execution checklist current and report the first unchecked item when blocked.
- Use `talk-it-through` for unresolved approach decisions and unexpected out-of-task obstacles.
- Stop for declared decisions, unresolved implementation uncertainty, unexpected out-of-task
  obstacles, destructive database actions, and security boundaries.
- End after implementation and focused validation; review, README, final proof, completion
  recording, and PR creation happen in `task-review`, `review-fix-worker`, and `finish-task`.
- Keep `[>]` distinct from `[x]`; only `sync-main` closes a merged task.
