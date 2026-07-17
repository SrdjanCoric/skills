---
name: to-plan
description: Turn a PRD, decision document, or conversation into the smallest independently verifiable vertical tasks under plans/tasks/, append them to the project's single local master plan, and wait for user approval before writing. Use to plan features, tracer-bullet work, or agreed decisions for implement-next-task.
---

# To Plan

Turn the source into self-contained local task files and register them in the project's single
master plan. Each task is the smallest complete behavior that can be implemented, verified, and
merged independently on its own branch.

## Process

### 1. Confirm the source

Use a PRD, decision document, or the current conversation. If the source is unclear, ask the user
to identify it. The source type never determines the number of tasks.

### 2. Locate or bootstrap the master plan

Find the master plan through the most specific `AGENTS.md` `Active plan` entry, falling back to
`CLAUDE.md`.

- If it exists, append tasks to it. Do not create another plan.
- Change an architectural-decision bullet only when the source contains a new durable decision.
- If no plan exists, explore the codebase first, create the master plan from the template below,
  and register it in `AGENTS.md` and in `CLAUDE.md` when that file exists.
- On an existing codebase, describe the current architecture and begin the task list with work
  planned now. Do not backfill shipped work.

### 3. Explore the codebase

Inspect the current architecture, domain vocabulary, patterns, integration layers, tests, and
relevant decision records.

Invoke `software-repository-guidelines` in `scope` mode with the source, repository state, and
proposed work. Load only relevant references. Map current-task requirements to the task where they
naturally apply. Do not turn unrelated repository hardening into feature work.

### 4. Draft the smallest vertical tasks

Break the source into tracer-bullet tasks.

<vertical-slice-rules>

- Create one task for each smallest complete, independently verifiable behavior.
- Cross every layer relevant to that behavior, including its tests or verification. Do not invent
  schema, API, or UI work when the behavior does not require it.
- Keep tests and verification in the task whose behavior they prove. Never defer them to a later
  horizontal task.
- Make every completed task demoable or verifiable on its own.
- Apply the split test repeatedly: list the task's observable behaviors. If one can be delivered
  and verified independently, split it into another vertical task. Continue until removing any
  behavior would make the remaining task incomplete.
- Keep each task small enough to understand, implement, verify, and review in one fresh
  implementation context. If it does not fit comfortably, apply the split test again. Do not split
  by technical layer to satisfy this limit.
- Give every task its own `feature/<kebab-slug>` branch and one PR. One PR is a ceiling, not a
  sizing target.
- Do not include volatile file names or line numbers. Include durable decisions such as routes,
  schema shapes, public contracts, and domain model names.

</vertical-slice-rules>

#### Wide mechanical refactors

Use a non-vertical refactor task only when a cross-cutting mechanical change cannot land green as
a vertical slice. Plan it as expand, migrate, and contract tasks:

1. Add the new form alongside the old.
2. Move callers in the smallest green batches the blast radius permits.
3. Remove the old form after every caller has migrated.

Each intermediate task must be mergeable and green. Do not create cleanup or prefactoring tasks
merely because they would be convenient.

#### Dependencies

Give each task the minimal set of direct predecessors whose merged output it needs. Use `none` when
it needs no predecessor. Eligibility comes from dependencies, not list position: every dependency
must be `[x]` before the task can start.

- Serialize tasks that change the same durable shared artifact by adding a direct dependency and
  naming the artifact in the dependent task.
- If a new task becomes a prerequisite of an unfinished existing task, update the existing task's
  `Depends on` field and master-plan pointer. Do not renumber stable task identifiers.

### 5. Confirm the breakdown before writing

Present every proposed task as a numbered list. For each task show:

- title;
- the independently verifiable outcome;
- adjacent behavior deliberately excluded;
- direct dependencies;
- automated verification, or the required manual verification;
- why another split would make the task incomplete.

Ask whether tasks should be split, merged, reordered, or re-scoped. Wait for explicit approval
before creating or modifying any plan or task file, including when only one task is proposed.

### 6. Separate implementation work from human checkpoints

- **Implementation work** is everything the agent can implement and verify autonomously. Use the
  `tdd` skill and phrase behavior work test-first where appropriate.
- **Human checkpoints** are limited to:
  - `[decision]` for an unresolved product, architecture, or scope choice, handled through
    `talk-it-through`;
  - `[verify]` when the result cannot be verified automatically;
  - `[confirm-db]` for real, shared, destructive, persistent, or ambiguous database or data work;
  - `[confirm-security]` for changes to authentication, authorization, sessions, secrets,
    cryptography, dependency trust, CI, sandboxing, or another trust boundary.

Automate verification whenever possible. A `[verify]` item must state why automation is impossible,
the exact steps the user must perform, the expected result, and what indicates failure. Manual
verification blocks task completion until the user confirms it passed.

### 7. Write the approved task files

Use the next ordinal after the maximum ordinal in `plans/tasks/` and `plans/tasks/done/`, padded to
four digits. Write `plans/tasks/NNNN-<kebab-slug>.md`.

Each task must contain the relevant source requirements and acceptance criteria so a fresh
implementer does not need the PRD or neighboring tasks. Point to durable decisions instead of
duplicating them.

For Software Repository Guidelines, name only the relevant reference files, applicable
requirements, and expected proof.

<task-file-template>
# Task NNNN: <Title>

**Branch**: `feature/<kebab-slug>`
**Depends on**: <direct predecessor ordinals, or `none`>
**Source**: <PRD, decision document, or conversation date> · **User stories**: <list>

## What to build

<One smallest complete, independently verifiable behavior described end to end.>

## Software Repository Guidelines

**Applicable references**: <only relevant references>

- [ ] <applicable requirement and expected repository, command, or CI proof>

## Implementation work

- [ ] <work item>

## Human checkpoints

- [ ] [decision] <question requiring shared understanding> (`talk-it-through`)
- [ ] [verify] <manual steps> · Expected: <result> · Failure: <failure signal> · Reason:
      <why automation is impossible>
- [ ] [confirm-db] <database or data action requiring approval>
- [ ] [confirm-security] <trust-boundary action requiring approval>

(Omit `Human checkpoints` when none apply.)

## Acceptance criteria

- [ ] <criterion proving the behavior>
</task-file-template>

### 8. Append task pointers

Append one pointer per approved task to the master plan. Mirror direct dependencies in an
`(after ...)` suffix and omit the suffix for `none`:

```markdown
- [ ] NNNN · <Title> → tasks/NNNN-<slug>.md
- [ ] NNNN · <Title> (after NNNN[, NNNN]) → tasks/NNNN-<slug>.md
```

Tell the user which task files were created and where they sit in the plan.

## Master-plan template

<master-plan-template>
# Plan: <Project Name>

> Source: <brief identifier or link>

This is the project's local master plan. Task bodies live in `plans/tasks/`; merged tasks move to
`plans/tasks/done/`.

## Workflow

- `to-plan` adds approved self-contained task files and pointers.
- `implement-next-task` takes the first eligible task, claims it as `[~]`, implements it through
  `tdd`, uses `talk-it-through` when the task or an unexpected obstacle requires a decision,
  updates the README when the current application state changed, runs `task-review`, proves the
  behavior, and invokes `create-pr` after user approval.
- `[ ]` means ready, `[~]` in progress, `[>]` complete with a CI-green PR awaiting merge, and `[x]`
  merged into `main`.
- `sync-main` verifies and merges the PR, synchronizes local `main`, cleans the merged branch,
  changes `[>]` to `[x]`, and moves the task to `tasks/done/`.
- A task is eligible only when every ordinal in its `(after ...)` list is `[x]`.
- Run one `implement-next-task` workflow at a time in the current checkout.

## Architectural decisions

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...

---

## Tasks

- [ ] 0001 · <Title> → tasks/0001-<slug>.md
- [ ] 0002 · <Title> (after 0001) → tasks/0002-<slug>.md
</master-plan-template>
