---
name: task-review
description: Review a completed task branch with one independent parallel review panel and write a self-contained findings todo-list under reviews/. Retains every review pass until finish-task removes the task's review files. Never remediates; the user runs review-fix-worker for fixes. Use standalone after implement-next-task.
---

# Task Review

Review the diff between `HEAD` and a fixed base with independent Standards, Spec, conditionally
applicable Bug, and conditional Security lenses. Run the panel once, with each active lens in its
own subagent in parallel. Write every accepted finding to a todo-list file under `reviews/` in the
repository. Do not fix findings here: remediation is the manual `review-fix-worker` loop, and a
clean or fully resolved review unblocks `finish-task`.

Invocation authorizes creating workflow files under `reviews/` and adding the `reviews/` ignore
entry to `.gitignore` when missing. Never delete or overwrite a review file; `finish-task` removes
all review files for the completed task. Never modify the implementation under review. Never write
any other review document or create review scaffolding outside `reviews/`.

## Inputs

Accept these optional inputs:

- `base`: the fixed point for a three-dot diff, such as `main`, a branch, or a commit;
- `spec`: the originating task, plan, or specification as a path or verbatim text;
- `task`: the local task-file path when one exists;
- `change-class`: `plan-only`, `documentation-only`, `dependencies`, `configuration`, `code`, or `mixed`;
- `validation-tier`: `documentation`, `focused`, or `canonical`.

When invoked standalone:

1. Ask for `base` if it is absent.
2. Resolve `spec` from a supplied path, issue reference in commits, or matching plan or spec file.
3. If `change-class` or `validation-tier` is absent, inspect the fixed base and complete current
   diff, derive both values directly, and fail closed to `mixed` plus `canonical` when ambiguous.
4. If no spec exists, skip the Spec lens and report that fact in the final result.

## Review workflow

### 1. Inspect prior review files

Before reviewing, inspect every `reviews/*.md` file without modifying it. For files whose
`**Branch:**` matches the current branch:

- when any finding is `open`, stop, report the file and open finding identifiers, and tell the user
  to run `/skill:review-fix-worker` (or explicitly skip or accept the remaining findings);
- when no finding is `open`, retain the file as review history and continue with a new review pass.

Leave files for other branches untouched. Never delete resolved review files here. `finish-task`
removes every review file associated with the selected task only after the task reaches a CI-green
PR.

### 2. Resolve the diff

Preflight the tree before resolving anything:

```sh
git status --porcelain
git rev-list --count <base>..HEAD
```

- Uncommitted tracked or untracked task work present → stop. The panel reviews committed work
  only, because the review file records a fixed head SHA. Tell the user to commit the task work on
  the feature branch first (`implement-next-task` and `review-fix-worker` both end with a commit;
  if their work is uncommitted, commit it or re-run the failing step), then re-run `task-review`.
  Never commit on the user's behalf and never review the working tree.
- Zero commits between base and HEAD → stop and report the empty diff.

Confirm the base resolves. Capture:

```sh
git diff <base>...HEAD
git log <base>..HEAD --oneline
```

Stop on a bad base or empty diff. Record the branch and head SHA. Keep the base fixed throughout
review.

### 3. Resolve repository requirements

Confirm or derive the change class and validation tier from the actual diff. Do not accept a
non-coding class when production or test code is present. Review current-task requirements and
established repository standards only. Report unrelated pre-existing problems as out of scope
findings and immediately mark them `deferred-out-of-scope`.

### 4. Decide whether Security runs

Run the Security lens when paths or diff content touch authentication, authorization, sessions,
secrets, tokens, input parsing, deserialization, SQL, shell or subprocess execution, network I/O,
file uploads, cryptography, sandboxing, CI trust boundaries, or prompt and guardrail code.

Otherwise skip Security and include `security_status: skipped-no-relevant-surface` in the final
result and the review file. A skip is not a security pass.

### 5. Run one independent review panel in parallel

For `plan-only` and `documentation-only` work, run Standards and Spec only. Add Bug only when the
diff contains executable examples, generated navigation, or another behavior-bearing documentation
surface. For other classes, run Standards, Spec, and Bug. Security remains conditional on the
surface rules above. Run each active lens in its own subagent, launching all active lenses in one
parallel batch (up to four subagents), and require only a JSON Finding array in response.

| Lens | Review target |
|------|---------------|
| Standards | Documented repository standards and meaningful tests |
| Spec | Missing, partial, incorrect, or out-of-scope behavior against the task or spec |
| Bug | Correctness, efficiency, and unnecessary complexity through the environment's code-review capability |
| Security | Exploitable trust-boundary failures through the environment's security-review capability |

Tell each subagent to inspect the diff directly, avoid invoking any task-review skill, avoid
spawning more subagents, and return findings with concrete evidence.

This is the only broad panel pass. Do not launch these lenses again.

### 6. Normalize and freeze findings

Dedupe identical findings while retaining every lens that reported them. Reject unsupported claims
that do not satisfy the Finding schema. Separate findings into:

- non-security findings caused by the current diff;
- security findings;
- unrelated pre-existing problems, which are recorded as `deferred-out-of-scope`.

Assign each accepted finding a stable identifier. Freeze this set. This skill never remediates, so
security findings are written as `open` like all others; the user decides fix-or-accept per finding
inside `review-fix-worker`.

### 7. Write the review file

Ensure `reviews/` exists at the repository root and is ignored by git: check `.gitignore` for a
`reviews/` entry and add that single line when missing. If the entry cannot be added, stop and ask
the user before writing findings.

Normalize any task path to its repository-relative path. Write one file per review pass. Use
`reviews/<task-file-stem>-<reviewed-head-short-sha>.md` when a task file exists, otherwise
`reviews/<branch-slug>-<reviewed-head-short-sha>.md`. Store the normalized path in `**Task:**`. The
file must be fully self-contained so `review-fix-worker` can start from a fresh context with no
other input:

```markdown
# Task review: <task title or branch>

**Repo:** <absolute repository root>
**Branch:** <branch>
**Base:** <base>
**Reviewed head:** <sha>
**Task:** <task-file path, or "none">
**Change class:** <change-class>
**Validation tier:** <validation-tier>
**Created:** <date and time>
**Lenses:** <lenses run, or "security: skipped-no-relevant-surface">

## Findings

### TR-1 — major — bug
**Status:** open
**Location:** path/to/file.py:42
**Claim:** One sentence describing the problem.
**Evidence:** The rule, spec text, or failing scenario that proves the claim.
**Suggestion:** Optional one-line fix direction.

### TR-2 — minor — standards
**Status:** open
...
```

Never overwrite an existing same-named file. If one exists for the same reviewed head, report that
this head was already reviewed and return its next action: `review-fix-worker` when it has open
findings, otherwise `finish-task` when no implementation commit has landed since that review.

## Finding schema

Every finding uses this shape:

```json
{
  "id": "TR-1",
  "axis": "standards | spec | bug | security",
  "severity": "blocker | major | minor | nit",
  "location": "path/to/file.py:42",
  "claim": "One sentence describing the problem.",
  "evidence": "The rule, spec text, or failing scenario that proves the claim.",
  "suggestion": "Optional one-line fix direction."
}
```

Evidence is mandatory:

- Standards findings cite the rule and its source.
- Spec findings quote the task or spec requirement.
- Bug and Security findings give a concrete input-to-outcome scenario.

Use `blocker` for a must-fix bug, security hole, or missing requirement; `major` for a hard standard
violation or partial requirement; `minor` for a judgment call; and `nit` for cosmetic issues.

## Lens briefs

### Standards

Give the subagent the diff command, commits, and repository standards files. Ask it to report
documented violations, missed current-task requirements, and tests that fail to verify real behavior
through public interfaces. Skip formatting and issues already enforced by tooling.

### Spec

Give the subagent the diff command, commits, and spec. Require a quote from the spec for each finding.
Report missing or partial requirements, scope creep, and incorrect behavior.

### Bug

Ask the subagent to run the environment's code-review capability on the diff. Require a concrete
failing scenario and stamp `axis: "bug"` on each finding.

### Security

Ask the subagent to run the environment's security-review capability on the diff. Require a concrete
attack or misuse scenario and stamp `axis: "security"` on each finding.

## Final in-context result

Return a concise structured result containing:

- base and reviewed head SHA;
- references and standards checked;
- security status;
- the frozen finding set with identifier, axis, and severity;
- prior review files inspected and retained;
- the new review file path;
- deferred out-of-scope concerns;
- the exact next command: `/skill:review-fix-worker` when any finding is `open`, otherwise
  `/skill:finish-task`.

A review with zero findings still writes the file with an empty `## Findings` section, so the
manual loop has an explicit clean signal. Do not report success when the review file could not be
written.
