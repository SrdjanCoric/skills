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
under `plans/tasks/`.

A pointer has **four states**, and they are not the same fact:

- `[ ]` — **todo**, unclaimed.
- `[~]` — **in progress**: an agent has claimed it (often in its own worktree); selection skips it.
- `[>]` — **done, PR open, awaiting merge**: the agent finished and opened the PR, but the code is
  **not on `main` yet**. The task file is still under `plans/tasks/`.
- `[x]` — **merged to `main`**: the PR landed; `sync-main` flips `[>]→[x]` and moves the file to
  `plans/tasks/done/`.

A pointer may carry an `(after NNNN, …)` suffix listing its direct prerequisites. **A task is
selectable only once every one of those ordinals is `[x]` (merged)** — not merely `[>]`. This is the
crux: a dependent is built by branching off `main`, so its prerequisite's code must actually be on
`main` first. `[>]` (done but unmerged) therefore does **not** unblock dependents. The plan is
small, so you read it directly — no subagent mapping needed.

## Arguments

An optional task identifier — an ordinal (`0005`) or a task-file path. If given, work **that** task
instead of scanning — but only **if it is currently doable**: its pointer must be `[ ]` *and* have
every `(after …)` ordinal `[x]` (merged). If it isn't, **stop and report why** rather than proceeding:
it's already `[~]` (in progress), already `[>]`/`[x]` (done/merged), or **blocked** because a named
prerequisite is still open, in progress, or unmerged. Never build a blocked task on a `main` that
lacks its prerequisite. This rule is the same in both standalone and `--worktree` mode. If absent,
take the **first eligible pointer** in the master plan's `## Tasks` list — the first that is `[ ]`
(not `[~]`, `[>]`, or `[x]`) **and** has every ordinal in its `(after …)` suffix marked `[x]`. A task
whose deps aren't all `[x]` is **blocked** (a prerequisite is still open, in progress, or done-but-
unmerged) — skip it and take the next eligible one. If **no** pointer is eligible (every remaining
task is done or blocked by in-flight/unmerged work), report "everything is blocked by in-progress
work" and exit without changing the plan — but first, if a `[>]` (done, awaiting merge) is what's
blocking, note that its PR may already be merged and `sync-main` simply wasn't run: suggest running
`sync-main` to close it out, which would unblock its dependents. Find the master plan via `CLAUDE.md`
"Active plan"; only if no plan exists, ask the user.

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
2. **Pick the candidate and claim it immediately.** If an ordinal/path was given, the candidate is
   **that** task — but claim it only if it passes the eligibility check (`[ ]` with every `(after …)`
   ordinal `[x]`); if it doesn't, report why it isn't doable (in progress, already done, or blocked by
   an unmerged prerequisite) and exit without creating a worktree. Otherwise scan the main-checkout
   plan's `## Tasks` top-down for the first `[ ]` pointer whose every `(after …)` ordinal is `[x]`
   (merged; a pointer with no suffix is trivially ready). Because a dep that is `[~]`, `[>]`, or `[ ]`
   is not `[x]`, this skips any task waiting on running, done-but-unmerged, *or* unbuilt work — so the
   candidate's prerequisites are all on `main`. If the scan finds nothing, print **"no independent
   task"** and exit. Once you have a candidate, **flip its pointer `[ ]→[~]` right now**, before the
   safety check below, so a concurrent `--worktree` run can't pick the same candidate during the
   (slow) subagent round-trip.
3. **Run the parallel-safety check in a subagent** (keep the git/file reads out of the main context).
   Spawn one `Explore`/`general-purpose` subagent that returns **one short digest**: (a) **Stale
   claims** — for every *other* `[~]` pointer (exclude the one you just claimed; its worktree doesn't
   exist yet), is there a live worktree (`git worktree list`) and branch (`git branch --list
   <branch>`)? List any with neither as a *likely-stale claim to reset by hand* (report only — never
   auto-reset). (b) **Shared-artifact conflict (backstop)** — declared conflicts are already
   serialized by an `(after …)` edge at planning time, so an eligible candidate should be clear; as a
   backstop against a *missed* edge, compare the candidate's task file against every live `[~]` **and
   `[>]`** task (both are unmerged branches off `main`, so either can collide) and flag any that
   obviously modify the same durable artifact (a prompt, the rubric). (c) **Forgotten merge** — for
   every `[>]` pointer, is its branch actually merged into `main`? A merged `[>]` means `sync-main`
   was never run, so the task is wrongly blocking its dependents; list these as *run `sync-main` to
   close them out* (report only — flipping `[>]→[x]` requires the human-run `sync-main`, never an
   auto-flip here). If the digest flags a conflict on the candidate, **release the claim `[~]→[ ]`**,
   return to step 2 for the next eligible `[ ]`, and re-check; if every candidate is conflict-blocked,
   release and print "no conflict-free task" and exit. Surface stale claims and forgotten merges to
   the user regardless.
4. **Create the worktree on the task branch.** Update `main` in the main checkout (`git pull`), then
   from the task's `**Branch**` field run `git worktree add ../<repo>-<slug> -b <branch>` off it,
   where `<repo>` is the main checkout's directory name and `<slug>` is a short slug from the branch
   (matching the existing sibling-worktree convention). Reuse the worktree/branch if it already
   exists. This **replaces Workflow step 4**; all code work happens in the worktree.
5. **Run the rest of the workflow in the worktree.** Workflow step 1 reduces to *loading* the claimed
   task file (its selection and claim are already done in steps 2–3 above); Workflow steps 2–3 (recon
   subagent, confirm) and 5–11 run as usual; Workflow step 4 (branch creation) is replaced by the
   worktree created above. The pointer flip `[~]→[>]` happens in Workflow step 11 — **after `createpr`
   succeeds** — written to the main-checkout plan, not the worktree cwd. The `[>]→[x]` flip and the
   move to `tasks/done/` happen later: run `sync-main` from the worktree once the PR merges, to mark
   it merged and tear the worktree down (it removes the worktree, refreshes `main`, deletes the
   branch, and closes out the pointer).

## Workflow

1. **Load the task.** Read the master plan directly (it's small): take the
   `## Architectural decisions` header and the first **eligible** pointer — skipping `[x]` (merged),
   `[>]` (done, awaiting merge), `[~]` (in progress), and any whose `(after …)` deps aren't all `[x]`
   (blocked) — or the one named by the argument **only if it is doable** (see Arguments: a named task
   that's in progress, done, or dependency-blocked is reported and stopped, not built). **Claim it
   immediately:** flip its
   pointer `[ ]→[~]` in the master plan before any other work, so a concurrent run — worktree or
   not — skips it. *(With `--worktree`, selection and the claim are already done in Worktree-mode
   steps 2–3; here just load the already-claimed task file.)* Then read **that one task file**
   directly, in full — its `**Branch**` field, every
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

10. **Mark the task done in its file.** Check off the items in the task file, record what was built,
    key file paths, and every decision made; append a short implementation-log note. **Leave the
    pointer at `[~]` and the task file under `plans/tasks/`** — the code is done but there is no PR
    yet, so the task stays claimed, not `[>]`.

11. **Get approval, open the PR, then flip the pointer.** When everything is finished — AFK done,
    tests green, manual verifications confirmed — ask the user to approve opening the PR. Send a push
    notification. Only after explicit approval, invoke the **createpr** skill to commit and open the
    PR from the branch; never open it before approval. **Once `createpr` succeeds, flip the pointer
    `[~]→[>]`** (done, PR open) in the master plan — in worktree mode at the resolved main-checkout
    path. If approval never comes, the pointer stays `[~]` (still claimed) rather than a false `[>]`
    with no PR. The task file stays under `plans/tasks/`; after the PR merges, `sync-main` closes it
    out — `[>]→[x]`, move to `tasks/done/`, tear down any worktree.

## Notification rule

Any time the workflow pauses for human input — a talk-it-through session, a `[verify]` request, or
final PR approval — call the `PushNotification` tool with a one-line summary of what is needed,
so the user gets it on their phone via Remote Control. Then wait for their response.

## Rules

- One task per run, in full — all of its items, not just the next one.
- All work happens on the feature branch, never on `main`.
- Selection always skips `[x]` (merged), `[>]` (done, awaiting merge), `[~]` (in progress), and
  **dependency-blocked** pointers (any `(after …)` ordinal not yet `[x]`). Every run claims its task
  `[~]` on selection and flips `[~]→[>]` **after the PR opens**; `sync-main` flips `[>]→[x]` at merge.
  So concurrent runs — worktree or not — never collide, and a dependent never starts until its
  prerequisite is on `main`. Worktree mode (`--worktree`) adds: claim the first not-blocked task
  *before* validating (release `[~]→[ ]` if conflict-blocked), run the parallel-safety subagent check
  (stale claims + shared-artifact conflict against `[~]` and `[>]` + forgotten merges), build in
  `../<repo>-<slug>`, and read/write the plan at the resolved main-checkout path.
- Independence here is **logical (build-order), not textual.** Eligibility guarantees a task's
  prerequisites are merged; it does not guarantee two unrelated parallel tasks won't edit the same
  file and merge-conflict. Genuinely shared durable artifacts (a prompt, the rubric) are serialized
  by an `(after …)` edge at planning time; everything else is an ordinary, resolvable conflict.
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
- On completion: open the PR via createpr (only after approval), then flip the pointer `[~]→[>]` and
  leave the file under `tasks/`. The move to `tasks/done/` and the `[>]→[x]` flip belong to
  `sync-main`, post-merge.
