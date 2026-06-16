---
name: implement-next-feature
description: Implement the next uncompleted feature (phase) from an implementation plan on its own feature branch — all AFK tasks autonomously, human-in-the-loop tasks via talk-it-through and manual verification, ending in a PR after user approval. Use when invoked with a plan file path, or when the user asks to build/implement the next feature from a plan.
---

# Implement Next Feature

Implement the **next uncompleted feature** of the plan in full on its own feature branch, then
open a PR once the user approves. The plan is the source of truth for what to build and what is
already done.

> The pre-delegation version of this skill is preserved verbatim at
> `original-pre-delegation-2026-06-12.md` in this folder; it is a resource file, not a loaded
> skill, so it never registers as a second command.

## Delegation principle

Keep the main agent's context small: delegate read-only reconnaissance to subagents, and only
ever delegate work that comes back as accurate as doing it yourself. Whole-plan reading and
disk/git verification are delegated (steps 1 and 2). What stays in the main agent: the
**verbatim** text of the active phase and the global conventions, every judgment call, the TDD
implementation, and any file you are about to edit (editing needs exact content). When you
delegate, the subagent must return the load-bearing parts **verbatim**, never summarized.

## Argument

A single argument: the path to the plan file (e.g. `plans/job-tracker-plan.md`). If no argument
is given, use the active plan named in the project's `CLAUDE.md` (look for an "Active plan"
entry). Only if no argument is given *and* the project defines no active plan, ask the user
which plan to use before doing anything else.

## Workflow

1. **Map the plan via a subagent (keep main context small).** Spawn one `Explore` (or
   `general-purpose`) subagent to read the entire plan file end to end and return, in one
   structured result: (a) the global ground rules, working conventions, and architectural
   decisions **verbatim**; (b) the **first phase that still has unchecked `- [ ]` tasks**,
   reproduced **verbatim in full** — its `**Branch**` field, every task with its
   AFK/`[decision]`/`[verify]` marker, and any phase-specific notes; (c) the paths of any
   decision docs that phase references; (d) a one-paragraph digest of the remaining phases and
   the implementation log, enough to know what is already done and to avoid pulling later work
   forward. You implement against the verbatim active phase and conventions, so nothing
   load-bearing is summarized away, and the other phases never enter main context. Read a
   referenced decision doc directly only when a task depends on it.

2. **Verify reality via a subagent.** Spawn an `Explore` subagent to check the plan's
   implementation-log claims for the active phase against git history and the actual files on
   disk, returning a short **drift report**: what the log says is done versus what the code
   shows, the paths of the files the phase touches, and the current branch and HEAD. The log
   can be stale or wrong; trust the code over the log. When you implement, read the specific
   files you will edit directly — editing needs exact content, so that part is never delegated.

3. **Pick the next feature.** Confirm the active phase the mapping step surfaced (the first
   phase with any unchecked `- [ ]` tasks). State clearly which feature you are implementing
   before you start.

4. **Create or check out the feature branch.** The branch name comes from the phase's
   `**Branch**` field. If the branch already exists (locally or on the remote), check it out and
   continue where work left off. If it doesn't, update `main` and create the branch from it.
   All work for this feature happens on this branch.

5. **Resolve `[decision]` tasks via talk-it-through.** Before writing code that a decision affects,
   resolve every open `[decision]` task by invoking the **talk-it-through** skill and reaching
   mutual agreement. Since this needs the user, **send a push notification first** (see
   notification rule below). Record each agreed decision in the plan.

6. **Implement all AFK tasks.** Build every AFK task in the feature, autonomously, without
   pausing for input. Before writing any code, **load the `tdd` skill** (via the Skill tool)
   and follow its red-green-refactor loop for every code task — do not substitute an informal
   test-first approach. Follow every ground rule the plan states. Stay scoped to this one
   feature — don't pull work forward from later phases. Bias hard toward AFK: if something
   *can* be verified with a test, script, or automated check, do that instead of asking the
   user.

7. **Verify your own work.** Run the relevant tests/checks for everything you built and confirm
   they pass. Report failures honestly with output; never bend tests to pass.

8. **Generate the feature review (AFK), then stop — do not act on it.** Once all AFK work is done
   and green, invoke the **`feature-review`** skill with `base=main` and `spec=` the **verbatim
   active plan phase plus its referenced decision docs**. It fans out a review panel — Standards,
   Spec, `/code-review`, and `/security-review` when the diff touches security-relevant code — and
   writes `reviews/<featureName>-review.md`. Generating the file runs fully AFK; **applying its
   findings does not.** Produce the review and leave the working tree untouched by it. Do not fix,
   edit, refactor, or otherwise act on anything the review surfaces in this step, not even a finding
   that looks obviously correct or trivial. The review is output, not a to-do list you execute.

9. **Hand the review to the user for manual verification, and batch the `[verify]` checks.** Present
   every `[verify]` task to the user in one batch — what to check and how — **and point them at
   `reviews/<featureName>-review.md` to read and verify themselves.** Send a single push
   notification covering both, then wait. **The review is the user's to verify, not yours to
   action.** Do not apply any review finding until the user has read the review and told you which
   findings to fix; then apply only the fixes they approved. The same gate covers `[verify]`
   failures — fix on this branch only what the user directs. After applying approved fixes, re-run
   the review (it overwrites the file) and hand it back for verification again.

10. **Update the plan.** Check off all completed tasks (`- [x]`), record what was built, key file
    paths, and every decision made via talk-it-through. Leave the plan an accurate source of truth.

11. **Get approval, then open the PR.** When everything is finished — AFK tasks done, tests
    green, manual verifications confirmed, plan updated — ask the user to approve opening the
    PR. Send a push notification. Only after explicit approval, invoke the **createpr** skill
    to commit the work and open the PR from the feature branch. Never open the PR before
    approval.

## Notification rule

Any time the workflow pauses for human input — a talk-it-through session, a `[verify]` request, or
final PR approval — call the `PushNotification` tool with a one-line summary of what is needed,
so the user gets it on their phone via Remote Control. Then wait for their response.

## Rules

- One feature per run, in full — all of its tasks, not just the next one.
- All work happens on the feature branch, never on `main`.
- Bias hard toward AFK; interrupt the user only for `[decision]`, `[verify]`, and PR approval.
- The feature review is advisory and user-verified: generate it AFK, then stop. Never apply, edit,
  or act on a review finding until the user has manually verified the review and told you which
  findings to fix. Apply only approved fixes, then re-run the review.
- Batch human input where possible (one notification per pause, not one per item).
- Verify what's done against the code, not just the log.
- Delegate read-only reconnaissance (whole-plan reading, disk/git verification) to subagents to
  keep main context small; keep judgment, the verbatim active phase, the TDD implementation, and
  any file you edit in the main agent. Delegate only what returns as accurate as doing it yourself.
- Update the plan before finishing; PR only via createpr, only after approval.
