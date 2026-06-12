---
name: prd-to-plan
description: Turn a PRD into a multi-phase implementation plan using tracer-bullet vertical slices, saved as a local Markdown file in ./plans/. Use when user wants to break down a PRD, create an implementation plan, plan phases from a PRD, or mentions "tracer bullets".
---

# PRD to Plan

Break a PRD into a phased implementation plan using vertical slices (tracer bullets). Each phase is a feature, implemented on its own feature branch. Output is a Markdown file in `./plans/`.

## Process

### 1. Confirm the PRD is in context

The PRD should already be in the conversation. If it isn't, ask the user to paste it or point you to the file.

### 2. Explore the codebase

If you have not already explored the codebase, do so to understand the current architecture, existing patterns, and integration layers.

### 3. Identify durable architectural decisions

Before slicing, identify high-level decisions that are unlikely to change throughout implementation:

- Route structures / URL patterns
- Database schema shape
- Key data models
- Authentication / authorization approach
- Third-party service boundaries

These go in the plan header so every phase can reference them.

### 4. Draft vertical slices

Break the PRD into **tracer bullet** phases. Each phase is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
- Each phase IS a feature and will be implemented on its own feature branch. Name the branch when drafting the slice: `feature/<kebab-slug>` (e.g. `feature/job-import`)
- Slice sizing test: a slice should be the right size for ONE feature branch / one PR — demoable on its own, mergeable without waiting on later phases
- Do NOT include specific file names, function names, or implementation details that are likely to change as later phases are built
- DO include durable decisions: route paths, schema shapes, data model names
</vertical-slice-rules>

### 5. Break each phase into AFK and human-in-the-loop tasks

Decompose each phase into concrete to-dos in two buckets:

- **AFK tasks** — everything the agent can implement *and verify* autonomously: code, schema, migrations, tests, automated checks. This is the **default bucket**. AFK tasks are implemented by invoking the **tdd** skill. Phrase tasks so the test comes first where it makes sense.
- **Human-in-the-loop tasks** — only two legitimate kinds:
  - `[decision]` — needs mutual agreement before/while building (UX behavior, naming, API shape that could go multiple ways). Resolved by invoking the **talk-it-through** skill.
  - `[verify]` — genuinely cannot be verified automatically (visual polish, third-party dashboard, a real email arriving). Each must state *why* it can't be automated.

<afk-bias-rule>
Bias hard toward AFK. A task goes in human-in-the-loop only if it truly cannot be done or verified autonomously. If a verification can become a test, script, or automated check — make it an AFK task instead. A phase with zero human-in-the-loop tasks is a good phase. Don't manufacture decisions: if the PRD or architectural decisions already answer it, it's AFK.
</afk-bias-rule>

### 6. Write the plan file

Create `./plans/` if it doesn't exist. Write the plan as a Markdown file named after the feature (e.g. `./plans/user-onboarding.md`). Use the template below.

<plan-template>
# Plan: <Feature Name>

> Source PRD: <brief identifier or link>

## Workflow

Each phase is one feature, implemented on its own branch created from `main` when the phase starts. AFK tasks are executed autonomously by the implement-next-feature skill; `[decision]` tasks are resolved via the talk-it-through skill; `[verify]` tasks pause for manual confirmation.

## Architectural decisions

Durable decisions that apply across all phases:

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...
- (add/remove sections as appropriate)

---

## Phase 1: <Title>

**Branch**: `feature/<kebab-slug>`
**User stories**: <list from PRD>

### What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

### AFK tasks

- [ ] Task 1
- [ ] Task 2

### Human-in-the-loop tasks

- [ ] [decision] <question needing mutual agreement> (talk-it-through)
- [ ] [verify] <what to check manually> — <why it can't be automated>

(Omit this section if the phase has none — that's a good phase.)

### Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2

---

## Phase 2: <Title>

**Branch**: `feature/<kebab-slug>`
**User stories**: <list from PRD>

### What to build

...

### AFK tasks

- [ ] ...

### Human-in-the-loop tasks

- [ ] ...

### Acceptance criteria

- [ ] ...

<!-- Repeat for each phase -->
</plan-template>
