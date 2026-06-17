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
- Prefer many thin slices over few thick ones
- Each task IS a feature on its own branch. Name it when drafting: `feature/<kebab-slug>`
- Sizing test: right size for ONE branch / one PR — demoable, mergeable without waiting on later tasks
- Order tasks **forward-only**: a later task must never block an earlier one. The plan's order is
  the execution order, so the first unfinished pointer is always runnable on its own.
- Do NOT include specific file names or details likely to change as later tasks are built
- DO include durable decisions: route paths, schema shapes, data model names
</vertical-slice-rules>

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

Add one line per new task to the master plan's `## Tasks` list, in execution order:

```
- [ ] NNNN · <Title> → tasks/NNNN-<kebab-slug>.md
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
- `implement-next-task` takes the first unfinished pointer (or an explicit task argument), builds
  it on its branch — AFK via `tdd`, `[decision]` via `talk-it-through`, `[verify]` paused for
  manual confirmation — runs `task-review`, then opens the PR after approval. On completion it
  moves the task file to `tasks/done/` and flips its pointer to `[x]`.

## Architectural decisions

Durable decisions that apply across all tasks:

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...
- (add/remove sections as appropriate)

---

## Tasks

- [ ] 0001 · <Title> → tasks/0001-<slug>.md
</master-plan-template>
