---
name: task-review
description: Review and remediate a completed task branch with one independent review panel, one batched remediation, and targeted closure verification instead of full-panel reruns. Use from implement-next-task or standalone when review quality must be preserved without repeated review-fix-review cycles.
---

# Task Review

Review the diff between `HEAD` and a fixed base with independent Standards, Spec, conditionally
applicable Bug, and conditional Security lenses. Run the panel once. Batch supported findings into one remediation,
then close those findings with focused tests and targeted evidence checks. Do not rerun the full
panel automatically.

Invocation authorizes modifications to the current branch for non-security remediation. Never
write a review document or create review scaffolding.

## Inputs

Accept these optional inputs:

- `base`: the fixed point for a three-dot diff, such as `main`, a branch, or a commit;
- `spec`: the originating task, plan, or specification as a path or verbatim text;
- `task`: the local task-file path when one exists;
- `change-class`: `plan-only`, `documentation-only`, `dependencies`, `configuration`, `code`, or `mixed`;
- `validation-tier`: `documentation`, `focused`, or `canonical`.

When `implement-next-task` supplies the inputs, use them without asking. When invoked standalone:

1. Ask for `base` if it is absent.
2. Resolve `spec` from a supplied path, issue reference in commits, or matching plan or spec file.
3. If `change-class` or `validation-tier` is absent, inspect the fixed base and complete current
   diff, derive both values directly, and fail closed to `mixed` plus `canonical` when ambiguous.
4. If no spec exists, skip the Spec lens and report that fact in the final result.

## Review workflow

### 1. Resolve the diff

Confirm the base resolves. Capture:

```sh
git diff <base>...HEAD
git log <base>..HEAD --oneline
```

Stop on a bad base or empty diff. Record the branch and head SHA. Keep the base fixed throughout
review and remediation.

### 2. Resolve repository requirements

Confirm or derive the change class and validation tier from the actual diff. Do not accept a
non-coding class when production or test code is present. Review current-task requirements and
established repository standards only. Report unrelated repository-health problems as out of scope.

### 3. Decide whether Security runs

Run the Security lens when paths or diff content touch authentication, authorization, sessions,
secrets, tokens, input parsing, deserialization, SQL, shell or subprocess execution, network I/O,
file uploads, cryptography, sandboxing, CI trust boundaries, or prompt and guardrail code.

Otherwise skip Security and include `security_status: skipped-no-relevant-surface` in the final
result. A skip is not a security pass.

### 4. Run one independent review panel

For `plan-only` and `documentation-only` work, run Standards and Spec only. Add Bug only when the
diff contains executable examples, generated navigation, or another behavior-bearing documentation
surface. For other classes, run Standards, Spec, and Bug. Security remains conditional on the
surface rules above. Run each active lens in its own subagent and require only a JSON Finding array in
response.

| Lens | Review target |
|------|---------------|
| Standards | Documented repository standards and meaningful tests |
| Spec | Missing, partial, incorrect, or out-of-scope behavior against the task or spec |
| Bug | Correctness, efficiency, and unnecessary complexity through `/code-review` in Claude Code or `/review` in Codex |
| Security | Exploitable trust-boundary failures through `/security-review` in Claude Code or `$codex-security:security-diff-scan` in Codex |

Tell each subagent to inspect the diff directly, avoid invoking any task-review skill, avoid
spawning more subagents, and return findings with concrete evidence. Run the active lenses in parallel when the
environment supports it.

This is the only broad panel pass. Do not launch these lenses again during closure.

### 5. Normalize and freeze findings

Dedupe identical findings while retaining every lens that reported them. Reject unsupported claims
that do not satisfy the Finding schema. Separate findings into:

- non-security findings caused by the current diff;
- security findings;
- unrelated pre-existing problems, which remain out of scope.

Assign each accepted finding a stable identifier. Freeze this set before remediation. Targeted
verification may close or keep these findings open, but it must not become a new broad review.

If a new concern appears during remediation, handle it as follows:

- Fix a failing test or direct regression caused by the remediation inside the same batch.
- Record an unrelated or minor concern as deferred.
- Stop and ask the user when the concern is a new blocker, a new major issue, or requires a change
  to architecture, scope, or a trust boundary.

### 6. Resolve security findings

Before changing security-sensitive code, handle every security finding with the user. Explain in
plain English:

- the realistic scenario;
- the likely consequence;
- the exposure or probability;
- what the fix would change.

Wait for an explicit decision on each finding:

- **Fix**: add the approved finding to the remediation batch.
- **Accept**: record the user's reason in the local task file when one exists. Without a task file,
  return the accepted risk to the caller for the PR risk section.

Every unresolved security finding blocks completion. An accepted finding remains visible in the
final result.

### 7. Remediate once, as one batch

Fix every supported non-security finding attributable to the current diff and each security finding
the user approved. Load `tdd` only for defects in production or test code with an observable
behavioral seam. Never invoke it for plan-only, documentation-only, dependency-only, or declarative
configuration-only remediation. Group related coding findings into coherent red-green-refactor
cycles instead of alternating between review and implementation.

The remediation batch may include several edits and test runs. It remains one batch because the
finding set is fixed and no review lens is rerun.

Run focused tests and checks for the affected files while remediating. Do not run the full canonical
suite after each finding.

### 8. Verify the frozen findings

Perform targeted closure verification without spawning the review panel again:

1. For each finding identifier, inspect the changed call site and the evidence that originally
   supported the finding.
2. Confirm the original failing scenario is covered by a focused test, static check, or direct diff
   evidence.
3. Run the focused test batch once after all remediation is assembled.
4. Run one final check at the classified validation tier: document/plan consistency for
   `documentation`, affected dependency or configuration proof for `focused`, and the repository's
   canonical check for `canonical`.
5. Mark each finding `fixed`, `accepted-security-risk`, `deferred-out-of-scope`, or `unresolved`.

Do not search untouched parts of the diff for new minor findings during this step. Do not rerun the
Standards, Spec, Bug, or Security panel.

If targeted verification fails, continue correcting the same frozen findings within the remediation
batch and rerun only the failed focused proof. If a fix materially changes architecture, scope, or a
trust boundary, stop and recommend a fresh `task-review` invocation after this one ends. Do not
start it without user approval.

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

Return no document. Return a concise structured result containing:

- base and initial reviewed head SHA;
- remediated head SHA;
- references and standards checked;
- security status;
- the frozen finding set and final status of each finding;
- one panel pass and one remediation batch;
- focused proof and final validation command at the classified tier;
- accepted security risks and user reasons;
- deferred concerns and remaining blockers;
- whether a full review with `task-review` is recommended.

Do not report success while a frozen finding remains unresolved or a security finding lacks a user
decision.
