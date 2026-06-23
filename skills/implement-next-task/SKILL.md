---
name: implement-next-task
description: Implement the next uncompleted task from the project's master plan on its own feature branch — all AFK work autonomously, human-in-the-loop work via talk-it-through and manual verification, ending in a PR after user approval. Use when invoked with a task id/path, or when the user asks to build/implement the next task.
---

# Implement Next Task

Implement the **next uncompleted task** in full on its own feature branch, then open a PR once
the user approves. The master plan is the source of truth for order and status; each task's own
file is the source of truth for what to build.

> The pre-delegation version of this skill is preserved verbatim at
> `original-pre-delegation-2026-06-12.md` in this folder; it is a resource file, not a loaded
> skill, so it never registers as a second command.

## How the plan is structured

The project has one **master plan** (named in `CLAUDE.md` "Active plan"): a durable
`## Architectural decisions` header plus an ordered `## Tasks` checklist of pointers like
`- [ ] NNNN · Title → tasks/NNNN-slug.md`. Each pointer links to a **self-contained task file**
under `plans/tasks/`; finished tasks live in `plans/tasks/done/` with their pointer flipped to
`[x]`. A pointer can also be `[~]` — **in progress**: a task an agent has claimed (often in its own
worktree); selection skips it. The plan is small, so you read it directly — no subagent mapping
needed.

## Arguments

An optional task identifier — an ordinal (`0005`) or a task-file path. If given, work that task,
overriding plan order. If absent, take the **first available pointer** in the master plan's
`## Tasks` list — the first that is **neither done (`[x]`) nor in progress (`[~]`)**. A `[~]`
pointer means another agent has already claimed that task (often in a worktree), so skip past it to
the next available one. Find the master plan via `CLAUDE.md` "Active plan"; only if no plan exists,
ask the user.

An optional `--worktree` flag — build the task in a dedicated git worktree instead of the current
checkout (see **Worktree mode** below).

## Worktree mode (`--worktree`)

With `--worktree`, run the same workflow but build in an isolated worktree so the task can proceed
in parallel with other work. Everything here is project-agnostic — resolve paths from git, never
hardcode them.

1. **Find the main checkout** — the worktree that holds `plans/`. It's the first entry of
   `git worktree list`, equivalently the parent of
   `git rev-parse --path-format=absolute --git-common-dir`. Read and write the master plan and task
   files **there**, never the worktree cwd. (`plans/` is typically gitignored, so it exists only in
   the main checkout; the main checkout is the single source of truth regardless.)
2. **Claim the task first.** The moment it's selected, flip its pointer `[ ]→[~]` in the
   main-checkout plan — *before* creating the worktree — so any other agent (on the main checkout,
   in another worktree, or a fresh run) skips it and takes the next available task.
3. **Create the worktree on the task branch.** Update `main` in the main checkout first
   (`git pull`), then from the task's `**Branch**` field run
   `git worktree add ../<repo>-<slug> -b <branch>` off it, where `<repo>` is the main checkout's
   directory name and `<slug>` is a short slug from the branch (matching the existing
   sibling-worktree convention). Reuse the worktree/branch if it already exists. This **replaces
   step 4**; all code work happens in the worktree.
4. **On completion**, flip the pointer `[~]→[x]` in the main-checkout plan (not `[ ]→[x]`) and open
   the PR as usual. A stale `[~]` left by an abandoned run is reset to `[ ]` by hand. After the PR
   merges, run `sync-main` from the worktree to tear it down (it removes the worktree, refreshes
   `main`, and deletes the branch).

## Workflow

1. **Load the task.** Read the master plan directly (it's small): take the
   `## Architectural decisions` header and the first **available** pointer — skipping `[x]` (done)
   and `[~]` (in progress) — or the one named by the argument. **Claim it immediately:** flip its
   pointer `[ ]→[~]` in the master plan before any other work, so a concurrent run — worktree or
   not — skips it. Then read **that one task file** directly, in full — its `**Branch**` field, every
   AFK/`[decision]`/`[verify]` item, and acceptance criteria. Read a decision doc the task
   references only when an item depends on it. The task file is self-contained, so the PRD and the
   other tasks never enter context.

2. **Verify reality — delegate the research to a subagent; do not sweep the codebase inline.**
   The recon sweep is the single biggest context sink, so it runs in a subagent's context, not the
   main thread. Spawn an `Explore` (or `general-purpose`) subagent and have it return **one
   structured digest**: a drift report (what the task's log/claims say vs what the code shows), the
   exact files and line ranges each item touches, the test files plus the conventions a new test
   must match, and the minimal current-code excerpts you'll need to write each red test and each
   fix. Trust the code over the log. Read back only that digest — never the pile of files it read.
   The only files you open directly in the main agent are pulled **just-in-time in step 6**, one at
   a time, immediately before you edit each, because Edit needs the exact current content. If the
   digest is thin, send the subagent back for more rather than reading the files yourself.

3. **Confirm the task.** State clearly which task you're implementing (ordinal + title) before you
   start.

4. **Create or check out the branch.** The name comes from the task's `**Branch**` field. If it
   exists (local or remote), check it out and continue. Otherwise update `main` and branch from it.
   All work happens on this branch. *(With `--worktree`, this is handled by Worktree mode — create
   the worktree on this branch instead of checking out here.)*

5. **Resolve `[decision]` items via talk-it-through.** Before writing code a decision affects,
   resolve every open `[decision]` item by invoking the **talk-it-through** skill and reaching
   agreement. This needs the user, so **send a push notification first**. Record each agreed
   decision in the task file.

6. **Implement all AFK work.** Build every AFK item, autonomously, without pausing. Before writing
   any code, **load the `tdd` skill** and follow its red-green-refactor loop for every code item —
   no informal substitute. Follow every ground rule in the architectural header. Stay scoped to
   this one task. Bias hard toward AFK: if something *can* be verified with a test or automated
   check, do that instead of asking.

7. **Verify your own work.** Run the relevant tests/checks and confirm they pass. Report failures
   honestly with output; never bend tests to pass.

8. **Generate the task review (AFK), then stop — do not act on it.** Once all AFK work is green,
   invoke the **`task-review`** skill with `base=main` and `spec=` the **verbatim task file plus
   its referenced decision docs**. It fans out a panel — Standards, Spec, `/code-review`, and
   `/security-review` when the diff touches security-relevant code — and writes
   `reviews/<taskName>-review.md`. **`task-review` derives `<taskName>` from the branch, which omits
   the ordinal — so prefix the task number onto the review file: ensure it lands at
   `reviews/NNNN-<taskName>-review.md` (e.g. `reviews/0031-guardrail-terminate-manipulation-review.md`),
   renaming `task-review`'s output if needed, so reviews sort by task and tie back to the plan.**
   In worktree mode the review is generated and written **in the worktree cwd** — it describes that
   branch's diff and `reviews/` is branch-scoped, so it stays with the branch, unlike the shared
   plan/task bookkeeping (step 10) that goes to the main checkout.
   Generating the file is AFK; **applying its findings is not.** Leave the working tree untouched by
   it. Do not fix, edit, or act on anything it surfaces here, not even an obviously-correct finding.
   The review is output, not a to-do list.

9. **Hand the review to the user, and batch the `[verify]` checks.** Present every `[verify]` item
   in one batch — what to check and how — **and point them at `reviews/NNNN-<taskName>-review.md` to
   read and verify themselves** (in worktree mode, give the full worktree path, since it isn't in
   their main checkout). Send a single push notification, then wait. Do not apply any
   review finding until the user has read the review and told you which to fix; then apply only the
   approved fixes. The same gate covers `[verify]` failures. After applying approved fixes, re-run
   the review (it overwrites the file) and hand it back.

10. **Complete the task.** Check off the items in the task file, record what was built, key file
    paths, and every decision made; append a short implementation-log note in the task file.
    **Move the task file from `plans/tasks/` to `plans/tasks/done/`**, and in the master plan flip
    its pointer `[~]→[x]` and repoint the link to `tasks/done/`. In worktree mode, write these at
    the resolved main-checkout path. Leave both an accurate source of truth.

11. **Get approval, then open the PR.** When everything is finished — AFK done, tests green,
    manual verifications confirmed, task moved to done, plan updated — ask the user to approve
    opening the PR. Send a push notification. Only after explicit approval, invoke the **createpr**
    skill to commit and open the PR from the branch. Never open the PR before approval.

## Notification rule

Any time the workflow pauses for human input — a talk-it-through session, a `[verify]` request, or
final PR approval — call the `PushNotification` tool with a one-line summary of what is needed,
so the user gets it on their phone via Remote Control. Then wait for their response.

## Rules

- One task per run, in full — all of its items, not just the next one.
- All work happens on the feature branch, never on `main`.
- Selection always skips both `[x]` (done) and `[~]` (in progress) pointers. Every run claims its
  task `[~]` on selection and flips `[~]→[x]` on completion, so concurrent runs — worktree or not —
  never collide. Worktree mode (`--worktree`) adds only: build in `../<repo>-<slug>` and read/write
  the plan at the resolved main-checkout path.
- Bias hard toward AFK; interrupt the user only for `[decision]`, `[verify]`, and PR approval.
- The task review is advisory and user-verified: generate it AFK, then stop. Never apply, edit, or
  act on a review finding until the user has manually verified the review and told you which to
  fix. Apply only approved fixes, then re-run the review.
- Batch human input where possible (one notification per pause, not one per item).
- Verify what's done against the code, not just the log.
- Research is delegated by default, not on a judgment call: the recon/understanding sweep always
  runs in a subagent that returns a digest, so the broad reading never lands in the main thread.
  Keep only judgment, the verbatim task file, that digest, the TDD implementation, and the one file
  you're editing right now in the main agent. The master plan itself is small — read it directly.
- On completion: move the task file to `tasks/done/` and flip its pointer before finishing; PR only
  via createpr, only after approval.
