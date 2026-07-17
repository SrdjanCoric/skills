---
name: task-review
description: Review and remediate a completed task branch against repository standards, the originating task or spec, correctness, and conditional security checks. Use from implement-next-task or standalone; invocation authorizes automatic fixes for current-diff non-security findings, while every security finding requires a plain-English user decision.
---

# Task Review

Review the diff between `HEAD` and a fixed base with independent Standards, Spec, Bug, and
conditional Security lenses. Fix supported non-security findings attributable to the current diff,
verify the fixes, and rerun the panel for at most two remediation passes.

Invocation authorizes modifications to the current branch for non-security remediation. Never
write a review document or create review scaffolding.

## Inputs

Accept these optional inputs:

- `base`: the fixed point for a three-dot diff, such as `main`, a branch, or a commit;
- `spec`: the originating task, plan, or specification as a path or verbatim text;
- `repository-guidelines`: the implement-mode result and proof when supplied by a caller;
- `task`: the local task-file path when one exists.

When `implement-next-task` supplies the inputs, use them without asking. When invoked standalone:

1. Ask for `base` if it is absent.
2. Resolve `spec` from a supplied path, issue reference in commits, or matching plan or spec file.
3. If no spec exists, skip the Spec lens and report that fact in the final in-context result.

## Review loop

### 1. Resolve the diff

Confirm the base resolves. Capture:

```sh
git diff <base>...HEAD
git log <base>..HEAD --oneline
```

Stop on a bad base or empty diff. Record the current branch and head SHA so remediation never
silently changes the review target.

### 2. Resolve repository requirements

Invoke `software-repository-guidelines` in `review` mode with the spec, actual diff, repository
state, supplied implement-mode result, and claimed proof. Select relevant references independently.
If the skill is unavailable, return a blocker finding instead of skipping it.

Review only current-task requirements and established repository standards. Do not turn unrelated
repository-health gaps into findings.

### 3. Decide whether Security runs

Run the Security lens when paths or diff content touch authentication, authorization, sessions,
secrets, tokens, input parsing, deserialization, SQL, shell or subprocess execution, network I/O,
file uploads, cryptography, sandboxing, CI trust boundaries, or prompt and guardrail code.

Otherwise skip Security and include `security_status: skipped-no-relevant-surface` in the final
result. A skip is not a security pass.

### 4. Run independent review lenses

Run each active lens in its own agent and require only a JSON Finding array in response.

| Lens | Review target |
|------|---------------|
| Standards | Documented repository standards, applicable Software Repository Guidelines, and meaningful tests |
| Spec | Missing, partial, incorrect, or out-of-scope behavior against the task or spec |
| Bug | Correctness, efficiency, and unnecessary complexity through `/code-review` in Claude Code or `/review` in Codex |
| Security | Exploitable trust-boundary failures through `/security-review` in Claude Code or `$codex-security:security-diff-scan` in Codex |

Tell every review agent to inspect the diff directly, avoid invoking `task-review`, avoid spawning
more agents, and return findings with concrete evidence. Run the active lenses in parallel when the
environment supports it.

### 5. Normalize findings

Dedupe identical findings while retaining every lens that reported them. Reject unsupported claims
that do not satisfy the Finding schema. Do not down-rank or discard a security finding to keep the
workflow moving.

Separate the result into:

- non-security findings caused by the current diff;
- security findings;
- unrelated pre-existing problems, which are reported as out of scope and never fixed here.

### 6. Resolve security findings

Before changing security-sensitive code, handle every new security finding with the user. Invoke
`write-well` and explain in plain English:

- what the risk is;
- how it could occur in a realistic scenario;
- the likely consequences;
- how exposed or probable it appears;
- what a fix would change.

Wait for an explicit decision on each finding:

- **Fix**: add the approved security finding to the remediation set.
- **Accept**: record the user's reason in the local task file when one exists. Without a task file,
  return the accepted risk to the caller so `create-pr` can include it in the PR `Risk` section.

Every unresolved security finding blocks completion. An accepted finding remains visible in the
final result but does not block a later pass when its evidence is unchanged.

### 7. Remediate and rereview

Fix every supported non-security finding attributable to the current diff without prompting the
user. Also fix security findings the user explicitly approved. Use `tdd` for behavioral defects
when a testable seam exists. Run the focused tests and checks required by the affected files.

Rerun the full applicable panel after remediation. Allow no more than two remediation and rereview
passes after the initial review.

- If the panel is clean apart from accepted security risks, return success.
- If a new security finding appears, return to the security decision step.
- If non-security findings remain after the second remediation pass, stop. Explain what remains,
  why the loop did not converge, and ask the user how to proceed.
- If a fix changes the branch head, record the new head SHA in the result.

## Finding schema

Every finding uses this shape:

```json
{
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
violation or partial requirement; `minor` for a judgment call; and `nit` for cosmetic issues. All
supported non-security severities are remediated automatically.

## Lens briefs

### Standards

Give the agent the diff command, commits, review-mode repository-guideline result, and repository
standards files. Ask it to report documented violations, missed current-task requirements, and
tests that fail to verify real behavior through public interfaces. Skip formatting and other issues
already enforced by tooling.

### Spec

Give the agent the diff command, commits, and spec. Ask it to report missing or partial
requirements, scope creep, and behavior built incorrectly. Require a quote from the spec for every
finding.

### Bug

Ask the agent to run the environment's code-review capability on the diff. Require concrete failing
scenarios and stamp `axis: "bug"` on every finding.

### Security

Ask the agent to run the environment's security-review capability on the diff. Require concrete
attack or misuse scenarios and stamp `axis: "security"` on every finding.

## Final in-context result

Return no document. Return a concise structured result containing:

- base and reviewed head SHA;
- references and standards checked;
- security status;
- findings fixed by axis and severity;
- remediation-pass count;
- tests and checks run;
- accepted security risks and user reasons;
- remaining blockers, if any.

Do not report success while an unresolved security finding or supported non-security finding
remains.
