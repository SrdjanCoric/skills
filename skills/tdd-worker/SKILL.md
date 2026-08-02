---
name: tdd-worker
description: Implement a claimed task's work items with the cost-aware TDD red-green-refactor loop and focused tier validation. Invoked by implement-next-task with a reconnaissance digest, or standalone with a task-file path when the branch and classification already exist.
---

# TDD Worker

Implement the `Implementation work` of one already-claimed task through test-driven development,
then run the focused pre-review validation for the recorded tier. This skill owns only building and
focused verification. It does not select or claim tasks, create branches, review, update the README,
record completion, or open PRs.

Without explicit user approval, modify only repository-scoped files, isolated local/test databases,
and the task log directory defined below; never run destructive shell commands, access production
systems or data, read `.env` files, or delete or move anything else outside the repository root.

## Inputs

When invoked by `implement-next-task`, accept and use without asking:

- `task`: the task-file path;
- `change-class`: `plan-only`, `documentation-only`, `dependencies`, `configuration`, `code`, or `mixed`;
- `validation-tier`: `documentation`, `focused`, or `canonical`;
- `tdd-applicable`: whether `tdd` applies;
- `recon`: the reconnaissance digest from the current-reality pass;
- `TASK_LOG_DIR`: the deterministic task log directory.

When invoked standalone:

1. Resolve the task from a supplied path, or find the single `[~]` task in the master plan. With
   zero or multiple `[~]` tasks and no supplied path, stop and report the state.
2. Confirm the current branch matches the task's `**Branch**` value. If not, stop and report the
   mismatch; never create or switch branches here.
3. Re-derive `change-class`, `validation-tier`, and `tddApplicable` from the task and current diff.
   Fail closed to `mixed` and `canonical` when ambiguity could hide executable behavior.
4. Read the task file and the relevant code and test ranges before editing. Trust the code over
   stale task notes.
5. Recompute `TASK_LOG_DIR` with the commands below and refuse mismatched or symbolic-link paths.

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

## Efficient evidence handling

Keep the main context for decisions, behavioral TDD evidence, targeted source inspection, and final
integration proof. Delegate broad, read-only interpretation to isolated subagents when available and
require concise structured results. Do not delegate primary RED-GREEN decisions, concurrent edits,
or stateful checks that share a database, port, simulator, fixture, generated output, or external
service.

Use the smallest evidence that proves a result. Broad commands and full test
output belong in the task-scoped log directory; return the exit status, duration, concise result,
relevant failure excerpt, and log path. For
searches, list matching files and counts first, then inspect matching lines only in selected files.
Batch independent tool calls in the same turn and read exact ranges instead of whole files when
possible.

Use stable names such as `unit.log`, `typecheck.log`, or `review-security.log` and overwrite a
superseded passing log instead of accumulating timestamped copies. Enforce the 5 MiB cap while each
command runs rather than writing an unbounded file and truncating it afterward. Retain the latest
output when truncation is necessary, and keep the task directory under 50 MiB by removing
superseded passing logs first. Never write secrets, credentials, environment contents, source files,
or database exports to these logs. Preserve current failure evidence until the failure is resolved.

## Implement the task

Load `tdd` only when the validated classifier result sets `tddApplicable: true` for `code` or
`mixed` work that changes observable coding behavior, then follow its cost-aware red-green-refactor
loop. Do not invoke `tdd` for plan-only, documentation-only, dependency-only, or declarative
configuration-only tasks. Verify those changes with targeted syntax, consistency, security,
configuration, or provider checks instead. Loading `tdd` alone does not complete a coding step:
observe RED and GREEN for each coherent behavior
cycle and retain the command evidence in working context. Implement every item under
`Implementation work`, follow the architectural decisions, and stay within the selected task.
Automate any verification that can be automated. Keep coherent bulk documentation, configuration,
localization, fixture, or workflow edits in one sequential editing pass when they would otherwise
require many reads and full-file writes. Delegate that pass to one sequential subagent only when isolated editing is available. Pass
`TASK_LOG_DIR` and a unique, stable log filename for delegated work. The main workflow reviews
its diff and owns integration decisions.

If implementation encounters an unexpected obstacle that the task does not address, stop and
return to the caller (or invoke `talk-it-through` when running standalone) before changing scope,
behavior, architecture, dependencies, or safety assumptions. Do not invent
requirements, reinterpret the task, or expand its scope to bypass an obstacle.

## Verify the implementation

Run validation according to the recorded tier and store verbose output in `TASK_LOG_DIR`:

- `documentation`: `git diff --check` plus available document, link, or plan-consistency checks;
  never lint, typecheck, build, or test the application solely for this class;
- `focused`: only checks that parse, validate, audit, or exercise the affected dependency or
  configuration surface;
- `canonical`: focused tests and affected-area checks while working; the one final canonical
  validation is owned by `finish-task` after the manual review loop.

Do not run unchanged canonical full validation repeatedly. Fix current-task gaps and leave
unrelated repository improvements alone.

When the diff is stable, inspect `main...HEAD` and the working tree, then classify the actual change
again and update the validation plan. Run independent read-only
unit/component/lint/typecheck checks only when they do not share mutable state. Isolated subagents may run independent checks in parallel when available. Pass `TASK_LOG_DIR` and a distinct stable log
filename to each job so parallel commands never write the same file. Each returns command, status,
duration, failure names, a short relevant excerpt, and log path. Never parallelize database, port, simulator, fixture, or generated
output checks unless their isolation is proved. Wait for every validation job to finish and resolve
its failures before returning.

## Commit and return result

Commit the completed work on the task branch before returning. Partial work may be committed as
far as it is green; never return with a silently dirty tree. If work is incomplete, say so in the
result and leave the tree committed at the last green state.

Return a concise structured result:

- task path, branch, and head SHA;
- work items implemented and any left incomplete, with reasons;
- RED/GREEN cycles observed, with command evidence summaries;
- classification of the actual diff and any change from the expected class;
- validation commands run, status, duration, and log paths;
- decisions made or obstacles deferred to the caller;
- whether the diff is stable and ready for manual `task-review`.

Do not report success while any work item is unimplemented or any focused validation job is
failing.
