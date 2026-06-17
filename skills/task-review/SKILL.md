---
name: task-review
description: Review a completed task branch with a panel of parallel agents — Standards (documented conventions + test quality), Spec (faithfulness to the originating task/plan), Bug (delegated to /code-review), and Security (delegated to /security-review, conditional). Synthesises all findings into one human-readable reviews/<task>-review.md via the write-well skill. Use at the end of implement-next-task (AFK) or standalone to review a branch, PR, or work-in-progress.
---

# Task Review

Fan out a panel of review agents over the diff between `HEAD` and a base, then synthesise their
findings into a single human-readable file at `reviews/<taskName>-review.md`.

Each lens runs as its **own agent** so contexts don't pollute each other. The lenses **find**;
a single author **writes**. The author runs the `write-well` skill so the report reads like a
person wrote it, not a linter dump.

This skill does **not** hunt bugs itself — it delegates that to `/code-review`. Its native value
is the two lenses that skill doesn't cover: **Standards** and **Spec**.

## The panel

| Lens | What it checks | How it runs |
|------|----------------|-------------|
| **Standards** | Does the diff follow this repo's *documented* coding standards, including whether the **tests** are meaningful (cover the spec's named edges, not bent/tautological/over-mocked)? | Inline brief (below) |
| **Spec** | Does the diff faithfully implement the originating spec — nothing missing, no scope creep, nothing mis-built? | Inline brief (below) |
| **Bug** | Correctness, efficiency, simplification | Invoke `/code-review` |
| **Security** | Injection, authz, secrets, unsafe I/O | Invoke `/security-review` — **conditional** |

## Inputs

This skill accepts two optional inputs so it works both AFK and standalone:

- `base` — the fixed point to diff against. Diff is always three-dot: `git diff <base>...HEAD`.
- `spec` — the originating contract (path or verbatim text).

**When a caller injects both (the AFK path, e.g. `implement-next-task`):** use them, ask
nothing — run fully unattended.

**When invoked standalone (`/task-review`):** self-resolve.
- `base`: if not given, ask "Review against what — a branch, a commit, or `main`?" Don't proceed
  without it.
- `spec`, in order: (1) issue refs in commit messages, fetched however the repo documents;
  (2) a path passed as an argument; (3) a PRD/plan under `plans/`, `docs/`, or `specs/` matching
  the branch/feature; (4) ask the user. If there genuinely is none, the Spec lens reports
  "no spec available" and is skipped.

## Process

### 1. Resolve the diff and `taskName`
Capture `git diff <base>...HEAD` and `git log <base>..HEAD --oneline`. Derive `taskName` from
the branch name with any leading `feature/` stripped (`feature/foo-bar` → `foo-bar`).

### 2. Ensure `reviews/` is gitignored
The review file is ephemeral, branch-scoped scaffolding. If the project's `.gitignore` doesn't
already ignore `reviews/`, append it. Never commit the review.

### 3. Decide whether Security runs
Scan the diff (paths + content) for security-relevant signals: auth/session, secret/token/key
handling, input parsing/deserialization, SQL/shell/subprocess, network or external I/O,
prompt/guardrail code, file uploads, crypto. **Any** hit → include the Security agent. Otherwise
skip it and record the skip in the report (see Synthesis). A silent skip must never read as
"security passed."

### 4. Fan out — one agent per lens, in parallel
Send a single message with one `Agent` call per active lens. Use `general-purpose` agents. Every
agent must return findings as a JSON array matching the **Finding schema** below — nothing else.

- **Standards** and **Spec**: use the inline briefs below.
- **Bug**: instruct the agent to invoke the `code-review` skill (default effort) on the diff.
  `/code-review` is a *prompt skill* — invoking it injects a playbook the agent must **execute
  inline**. The agent runs as a leaf, so it **cannot spawn its own sub-agents**; it must work the
  playbook itself rather than try to nest agents. Tell it to **return only the JSON Finding array**
  (the skill emits prose otherwise) and to stamp **`axis: "bug"` on every finding** regardless of
  the categories `/code-review` uses internally.
- **Security** (only if step 3 selected it): same pattern, invoking the `security-review` skill,
  stamping **`axis: "security"`** on every finding.

### 5. Synthesise via `write-well`
Spawn one synthesis author agent. Give it every lens's findings and instruct it to **load the
`write-well` skill** (via the Skill tool) and author the whole report through it. The author obeys
the Synthesis rules below and returns the finished markdown.

### 6. Write the file
Write the author's output to `reviews/<taskName>-review.md` (overwrite if it exists). Tell the
user where it is.

## Finding schema

Every finding, every lens, the same shape:

```json
{
  "axis": "standards | spec | bug | security",
  "severity": "blocker | major | minor | nit",
  "location": "path/to/file.py:42",
  "claim": "One sentence: what is wrong.",
  "evidence": "The citation backing the claim (see below).",
  "suggestion": "Optional one-line fix direction."
}
```

**`evidence` is mandatory — no finding without a citation:**
- Standards → the documented rule + where it's written (e.g. `CLAUDE.md`: "never use `any`").
- Spec → the quoted spec/plan line the code fails to meet.
- Bug / Security → the concrete failing scenario (inputs → wrong outcome).

**Severity:** `blocker` = must fix before merge (real bug, security hole, missing spec
requirement); `major` = hard standards violation or partial spec — should fix; `minor` = judgment
call; `nit` = cosmetic.

## Synthesis rules

- **Lead with a short human summary**: a sentence or two of plain prose, then counts per lens and
  the single worst finding.
- **Keep lenses as distinct labeled sections** — `## Standards`, `## Spec`, `## Bug`,
  `## Security`. Rank by severity **within** a section, **never across** — one lens must not mask
  another.
- **Dedupe allowed**: merge identical findings flagged by more than one lens, noting which.
- **No silent drops**: if you merge or demote a finding, say so. A finding never just disappears.
- **If Security was skipped**, add a one-line `## Security` note: "Skipped — no security-relevant
  surface touched." If a lens found nothing, say "No findings" — don't omit the section.
- Human voice throughout (that's why the author runs `write-well`). No filler, no restating the
  obvious, no inventing severity to look thorough.

## Lens briefs

**Standards agent** — give it the diff command, the commit list, and the repo's standards docs
(`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `CONTEXT.md`, `docs/adr/`, any `STYLE.md` /
`STANDARDS.md`, and config files like `eslint.config.*` / `tsconfig.json` / `biome.json`):

> Read the standards docs, then the diff. Report every place the diff violates a **documented**
> standard — cite the rule and where it's written. Separately, judge the **tests** in the diff: do
> they verify real behaviour through public interfaces, cover the edges the spec named, and avoid
> being bent/tautological/over-mocked? Skip anything tooling already enforces (formatting,
> lint-autofixable). Return only the JSON Finding array, `axis: "standards"`.

**Spec agent** — give it the diff command, the commit list, and the spec (path or text):

> Read the spec, then the diff. Report: (a) requirements the spec asked for that are missing or
> partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that
> look implemented but built wrong. Quote the spec line for each finding. Return only the JSON
> Finding array, `axis: "spec"`.

## Why a panel of separate agents

A change can pass one lens and fail another — clean conventions but the wrong behaviour, or exactly
what the spec asked but a security hole. Separate agents keep each judgment independent; keeping the
sections distinct in the report stops one from masking the rest.
