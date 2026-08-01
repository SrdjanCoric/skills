---
name: implement-next-task
description: Implement the next eligible local task with TDD, review, validation, documentation, and PR-quality gates while minimizing parent-context growth and avoidable wall time. Use with a task ordinal/path or when asked to build the next task.
---

# Implement Next Task

Implement one eligible task in full on its feature branch. Use one implementation workflow at a
time in the current checkout. The master plan controls order and status; the selected task file is
the contract for what to build.

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
- [ ] `tdd` completed for coding behavior, or correctly skipped for non-coding work
- [ ] Implementation work completed within scope
- [ ] Focused pre-review validation passed
- [ ] `task-review` completed without unresolved findings and supplied final validation at the classified tier
- [ ] README inspected after review; `write-well` audit completed only when README prose changed, or no-impact reason recorded
- [ ] Highest-level automated and required manual proof passed
- [ ] Task file records implementation, decisions, documentation, review, and proof
- [ ] User approved PR creation and the current head is CI-green

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
otherwise stop rather than writing into it. Keep the directory through implementation, PR creation,
and CI so failures remain diagnosable. `sync-main` removes it only after successful merge and
synchronization.

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

### 7. Verify the implementation

Run validation according to the recorded tier and store verbose output in `TASK_LOG_DIR`:

- `documentation`: `git diff --check` plus available document, link, or plan-consistency checks;
  never lint, typecheck, build, or test the application solely for this class;
- `focused`: only checks that parse, validate, audit, or exercise the affected dependency or
  configuration surface;
- `canonical`: focused tests and affected-area checks while working, with one final canonical
  validation owned by review.

Do not run unchanged canonical full validation repeatedly. Fix current-task gaps and leave
unrelated repository improvements alone.

When the diff is stable, inspect `main...HEAD` and the working tree, then classify the actual change
again and update the validation plan before review. Run independent read-only
unit/component/lint/typecheck checks only when they do not share mutable state. Isolated subagents may run independent checks in parallel when available. Pass `TASK_LOG_DIR` and a distinct stable log
filename to each job so parallel commands never write the same file. Each returns command, status,
duration, failure names, a short relevant excerpt, and log path. Never parallelize database, port, simulator, fixture, or generated
output checks unless their isolation is proved. Wait for every validation job to finish and resolve
its failures before invoking `task-review`; validation must never overlap review remediation.

### 8. Run task review

Invoke `task-review` with:

- `base=main`;
- `spec=` the verbatim task file and referenced decision documents;
- `task=` the task-file path;
- `change-class=` the actual diff classification;
- `validation-tier=` `documentation`, `focused`, or `canonical`.

`task-review` runs the lenses applicable to the change class, batches supported remediation, and
closes the frozen findings with targeted verification. It does not rerun the full panel after fixes
and writes no review document.

For every security finding, wait while `task-review` explains the risk and consequences
plainly. Continue only after the user approves a fix or accepts the risk. Pass accepted risks and
their reasons to `create-pr`.

If targeted verification recommends a fresh review because remediation materially changed
architecture, scope, or a trust boundary, stop and ask the user before starting another
`task-review` invocation.

### 9. Update the README

Inspect the README after task-review remediation, when the implementation has reached its reviewed
state. When the task changed current application behavior, setup, configuration, or usage, invoke
`write-well` only for the README update and audit only the affected prose. Complete the skill's full
audit loop; loading `write-well` alone does not complete this step. Describe the current application,
not a history of what changed. Record the affected sections and audit pass count in the task file.

Leave the README unchanged when the task has no documentation impact and record that conclusion in
the task file.

### 10. Prove the final behavior

Treat `task-review`'s post-remediation validation as the workflow's final validation for
the recorded tier. Do not rerun it. A documentation or focused tier must not be promoted to the
application's canonical suite without an actual diff reclassification or explicit acceptance
criterion. Run only the highest-level automated proof not already covered by that check, plus any
focused check invalidated by a later change. Prefer browser-level proof for
user-facing behavior and executable scripts or disposable local environments for CLI, API,
database, and provider workflows.

When automated verification is impossible, give the user exact steps, the expected result, and
the signal that indicates failure. Explain why the agent cannot perform the check, then wait for
the user to confirm it passed. Manual verification blocks completion.

### 11. Record completion

Check off the task's implementation work, human checkpoints, and acceptance criteria. Record what
was built, decisions made, relevant file paths, README disposition, automated proof, manual
verification, and accepted security risks. Leave the master-plan pointer at `[~]`
until a PR exists and its CI is green.

### 12. Open the PR after approval

Summarize the completed task and ask the user to approve opening the PR. Do not invoke `create-pr`
without explicit approval.

After approval, invoke `create-pr` with the task path, task-review result, verification
proof, and accepted security risks. `create-pr` opens or updates the PR and
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
- Run task-review before the one final README update.
- Stop for declared decisions, unresolved implementation uncertainty, unexpected out-of-task
  obstacles, impossible-to-automate verification, destructive database actions, security
  boundaries, review security findings, and PR approval.
- Verify completion against the code and observable behavior, not the implementation log.
- Keep `[>]` distinct from `[x]`; only `sync-main` closes a merged task.
