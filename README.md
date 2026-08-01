# Skills

My personal coding-agent skills. Each skill lives under `skills/` with a `SKILL.md` that defines when it applies and how it works.

## Installation

```bash
npx skills@latest add SrdjanCoric/skills
```

The installer lists every skill and lets you choose which ones to install. Run it again later to pull updates.

To install one skill directly:

```bash
npx skills@latest add SrdjanCoric/skills --skill write-well
```

The installer does not install a skill's dependencies automatically. Use the dependency section below when installing individual skills.

## Planning workflow

Use `talk-it-through` for a bounded planning question. It inspects the repository, resolves decisions, delegates factual research or consequential interface design when needed, and produces a durable planning brief. Use `map-it-out` instead when a large initiative needs a decision map that survives across sessions.

A typical path is:

```text
talk-it-through ─┐
                  ├→ write-planning-brief → write-a-prd (optional) → to-plan
map-it-out ───────┘
                                                        ↓
implement-next-task → create-pr → sync-main
```

`implement-next-task` runs `tdd` when behavior should be built test-first and runs `task-review` before preparing the PR.

## Skills

### Planning and specification

- **talk-it-through**: Resolves a bounded software-planning question through repository inspection, focused discussion, authoritative research, and divergent interface design when warranted. Produces a planning brief when the work is ready.
- **map-it-out**: Manually maps a large, uncertain initiative as durable decisions, dependencies, frontier, fog, and scope. Resume it across sessions until the initiative is ready for a planning brief.
- **research-for-planning**: Resolves bounded external factual questions through authoritative research and saves a compact artifact with claim-level citations and unresolved uncertainty.
- **design-it-twice**: Generates and compares divergent designs for a consequential, hard-to-reverse interface or seam before committing to one approach.
- **write-planning-brief**: Distills completed planning discussions and evidence into a durable, readiness-gated source document without reopening the interview.
- **write-a-prd**: Converts a planning brief or completed discussion into a product and implementation specification that preserves scenarios, decisions, testing seams, evidence, and unresolved checkpoints.
- **to-plan**: Splits an approved source into the smallest independently verifiable vertical tasks, records dependencies in the project's master plan, and waits for approval before writing files.

### Delivery workflow

- **implement-next-task**: Selects the next eligible local task, implements it with appropriate TDD and validation, runs task review, updates documentation, proves the behavior, and prepares a CI-green PR after approval.
- **tdd**: Uses cost-aware red-green-refactor cycles grouped around coherent behavior while preserving observable defect proof and highest-level journey verification.
- **task-review**: Runs one independent review panel, batches supported remediation, requires a user decision for every security finding, and verifies closure without repeatedly rerunning the full panel.
- **create-pr**: Commits and pushes the current unit of work, opens or updates its PR, handles in-scope CI failures, and stops only when the current PR head is green. It never merges.
- **sync-main**: Verifies and remotely merges a ready PR, fast-forwards local `main`, cleans the merged branch, and closes the matching local task.

### Supporting skills

- **diagnose**: Works hard bugs and performance regressions through reproduce, minimize, hypothesize, instrument, fix, and regression-test steps.
- **handoff**: Captures a focused bug, feature pivot, or context slice so a fresh agent can continue without rediscovering the expensive parts.
- **software-repository-guidelines**: Manually audits repository engineering health and returns evidence-backed hardening recommendations. It is explicit-only and never runs as a routine planning, implementation, review, or PR gate.
- **teach**: Maintains a persistent teaching workspace and produces one focused, interactive HTML lesson at a time, shaped by the learner's feedback.
- **write-well**: Writes and revises prose in a direct human voice using an adversarial audit for common machine-written patterns.

## Dependencies

Arrows show direct skill invocation. Conditional dependencies are marked with `when needed`.

```text
implement-next-task → talk-it-through, tdd, task-review, write-well, create-pr
create-pr           → write-well, diagnose (when CI fails)
task-review         → tdd (when code remediation needs behavioral proof)
talk-it-through     → research-for-planning (when needed), design-it-twice (when needed), write-planning-brief
map-it-out          → talk-it-through, research-for-planning (when needed), write-well, write-planning-brief
write-a-prd         → write-well
teach               → talk-it-through, write-well
diagnose            → handoff (when a fresh context is needed)
```

`task-review` also uses the host environment's code-review capability and, when the diff touches a relevant trust boundary, its security-review capability.

No skill automatically invokes `software-repository-guidelines`. `map-it-out` and `software-repository-guidelines` require explicit manual invocation.

`to-plan` names the skills used later in the task lifecycle but does not invoke them while writing the plan. `sync-main` is not an install-time dependency of `implement-next-task`; it is the separately authorized final step that merges the ready PR and closes the task.

To install the complete delivery workflow:

```bash
npx skills@latest add SrdjanCoric/skills \
  --skill implement-next-task \
  --skill talk-it-through \
  --skill research-for-planning \
  --skill design-it-twice \
  --skill write-planning-brief \
  --skill tdd \
  --skill task-review \
  --skill create-pr \
  --skill diagnose \
  --skill handoff \
  --skill write-well
```
