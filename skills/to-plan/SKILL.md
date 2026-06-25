---
name: to-plan
description: Turn a source (PRD, decision doc, or the current conversation) into one or more self-contained task files under plans/tasks/, appended as pointers to the project's single master plan. Use when the user wants to break work into tasks, plan a feature, turn a PRD/decision/chat into actionable work, or mentions "tracer bullets".
---

# To Plan

Turn a source — a PRD, a decision doc, or just the current conversation — into **task files**
and register them in the project's **master plan**. Each task is one vertical slice: a feature
on its own branch, ending in one PR. There is exactly **one master plan per project**; this
skill appends to it and never creates a second one.

## Process

### 1. Confirm the source is in context

The source can be a PRD, a decision doc, or the conversation so far. If you're unsure what to
plan from, ask the user to point you at it.

### 2. Locate (or bootstrap) the master plan

Find the project's master plan via the **`CLAUDE.md` "Active plan"** entry.

- **It exists** → you will *append* new task file(s) + pointer(s) to it. Do not recreate it. Do
  not rewrite its `## Architectural decisions` header unless a genuinely new durable decision
  emerged — if so, add/amend that one bullet only.
- **It does not exist** → bootstrap it (see the master-plan template), then register it in
  `CLAUDE.md` as the Active plan. On an **existing codebase**, derive the
  `## Architectural decisions` header from the *current* architecture (explore the code first —
  step 3), and start the `## Tasks` list from the work being planned **now**. Do **not** backfill
  already-shipped work as done tasks — the plan begins from here, with no history; the existing
  code is the record of what came before.

### 3. Explore the codebase

If you have not already, explore to understand the current architecture, patterns, and
integration layers — so tasks are grounded and the architectural header stays accurate.

### 4. Draft vertical slices

Break the source into **tracer bullet** tasks. Each cuts through ALL integration layers
end-to-end, not a horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- **Make each slice the SMALLEST it can be while staying a complete vertical slice.** The floor: it
  must still cut through all layers and be demoable on its own. Stay at that floor — every extra
  behavior folded in costs more to build and review.
- Split test: if a slice contains two independently demoable behaviors, split it into two slices
  ordered forward-only. Default to splitting.
- Each task IS a feature on its own branch. Name it when drafting: `feature/<kebab-slug>`
- Sizing test: the smallest demoable change mergeable without waiting on later tasks. One PR is the
  ceiling, not the target.
- Order tasks **forward-first**: edges normally point to lower ordinals, so the plan reads
  top-to-bottom as a sensible default sequence. But **eligibility — deps all `[x]` (merged), not list
  position — is the authority on what's runnable**: the back-patch rule below can add a backward edge
  (an existing task depending on a newer, higher-ordinal one), so do not rely on "the first unfinished
  pointer is always runnable." Ordinals are stable IDs, not a runnability guarantee.
- Do NOT include specific file names or details likely to change as later tasks are built
- DO include durable decisions: route paths, schema shapes, data model names
</vertical-slice-rules>

**Record dependencies.** After ordering, give each task a `Depends on` set: the **minimal** list of
direct predecessors whose output it actually builds on — not "everything before it." A task that
needs nothing prior is `none`. Edges normally point to lower ordinals, so the plan reads
top-to-bottom as a sensible default order. These edges are the dependency map: they let
`implement-next-task --worktree` find a task that isn't blocked by whatever is currently running, and
they — not list position — decide what's runnable.

Two more edge rules:

- **Serialize shared-artifact conflicts.** If two tasks both modify the same *durable* shared
  artifact — a prompt, a rubric, a shared module named in the architectural header — they would
  collide badly if built in parallel. Add an `after` edge between them so they run sequentially, and
  annotate the reason in the dependent's field: `Depends on: 0031 (shared: director-prompt)`. (The
  pointer suffix stays the plain `(after 0031)`.) This only applies to durable artifacts you can name
  at planning time; do not invent volatile file names.
- **Back-patch new prerequisites.** When a task you're adding now becomes a prerequisite of an
  already-listed *unfinished* task, update that existing task's `Depends on` field **and** its pointer
  `(after …)` suffix. This creates a **backward edge** — the existing lower-ordinal task now depends
  on the new higher one. That is allowed: eligibility, not ordinal order, governs selection, and the
  engine simply leaves the existing task blocked until the new prerequisite merges. Do **not** renumber
  to keep edges forward — ordinals are stable IDs (branches, `done/` files, links reference them).
  Appending without back-patching silently rots the graph.

For a multi-task source, **present the proposed edges to the user and confirm before writing** — per
the project's "don't assume" rule, the graph is not inferred silently.

From a fat PRD this yields several tasks; from a decision doc or a chat it's usually **one**.

### 5. Break each task into AFK and human-in-the-loop work

- **AFK tasks** — everything the agent can implement *and verify* autonomously: code, schema,
  migrations, tests, automated checks. The **default bucket**, implemented via the **tdd** skill.
  Phrase so the test comes first where it makes sense.
- **Human-in-the-loop tasks** — only two kinds:
  - `[decision]` — needs mutual agreement before/while building. Resolved via **talk-it-through**.
  - `[verify]` — genuinely cannot be verified automatically. Each must state *why*.

<afk-bias-rule>
Bias hard toward AFK. A task goes human-in-the-loop only if it truly cannot be done or verified
autonomously. If a verification can become a test, script, or automated check — make it AFK. A
task with zero human-in-the-loop items is a good task. Don't manufacture decisions the source or
architectural header already answers.
</afk-bias-rule>

### 6. Write the task file(s)

For each slice, write a **self-contained** task file. The next ordinal is `max(ordinal across
plans/tasks/ and plans/tasks/done/) + 1`, zero-padded to four digits. Filename:
`plans/tasks/NNNN-<kebab-slug>.md`.

Self-contained means the file carries everything the implementer needs — the relevant user
stories and acceptance criteria distilled from the source — so implementation never has to reach
back to the PRD. Reference durable decisions by pointing at the master plan header or a decision
doc; don't duplicate them.

<task-file-template>
# Task NNNN: <Title>

**Branch**: `feature/<kebab-slug>`
**Depends on**: <comma-separated ordinals of direct predecessors, or `none`>
**Source**: <PRD / decision doc / "talk-it-through <date>"> · **User stories**: <list>

## What to build

A concise description of this vertical slice — the end-to-end behavior, not layer-by-layer
implementation.

## AFK tasks

- [ ] Task 1
- [ ] Task 2

## Human-in-the-loop tasks

- [ ] [decision] <question needing mutual agreement> (talk-it-through)
- [ ] [verify] <what to check manually> — <why it can't be automated>

(Omit this section if the task has none — that's a good task.)

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
</task-file-template>

### 7. Append the pointer(s)

Add one line per new task to the master plan's `## Tasks` list, in execution order. Mirror the task's
`Depends on` onto the pointer as an `(after …)` suffix so the plan itself is the readable dependency
map; omit the suffix when `Depends on` is `none`:

```
- [ ] NNNN · <Title> (after NNNN[, NNNN]) → tasks/NNNN-<kebab-slug>.md
- [ ] NNNN · <Title> → tasks/NNNN-<kebab-slug>.md          # no prerequisites
```

Tell the user which task files you created and where they sit in the plan order.

---

## Master-plan template (bootstrap only — step 2)

<master-plan-template>
# Plan: <Project Name>

> Source PRD: <brief identifier or link>

This is the project's master plan: a durable architectural header plus an ordered list of task
pointers. Each task is one feature on its own branch, ending in a PR. Task bodies live in
`plans/tasks/`; finished tasks move to `plans/tasks/done/`.

## Workflow

- New work is added by the `to-plan` skill: a self-contained task file under
  `plans/tasks/NNNN-<slug>.md` plus a pointer below. It appends; it never creates a second plan.
- `implement-next-task` takes the first eligible pointer (or an explicit task argument), builds it
  on its branch — AFK via `tdd`, `[decision]` via `talk-it-through`, `[verify]` paused for manual
  confirmation — runs `task-review`, then opens the PR after approval and flips the pointer to `[>]`.
- A pointer has four states: `[ ]` todo · `[~]` in progress (claimed) · `[>]` done, PR open,
  awaiting merge · `[x]` merged to `main`. `sync-main` flips `[>]→[x]` and moves the task file to
  `tasks/done/` once the PR merges.
- Pointers carry their direct prerequisites as an `(after NNNN, …)` suffix (none = no suffix). A
  task is selectable only once every ordinal in its `(after …)` list is **`[x]` (merged)** — so a
  dependent never branches off `main` before its prerequisite is actually on `main`.
  `implement-next-task --worktree` uses this to pick the first task not blocked by in-progress work,
  or reports "no independent task" when every remaining task is blocked.

## Architectural decisions

Durable decisions that apply across all tasks:

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...
- (add/remove sections as appropriate)

---

## Tasks

- [ ] 0001 · <Title> → tasks/0001-<slug>.md
- [ ] 0002 · <Title> (after 0001) → tasks/0002-<slug>.md
</master-plan-template>
