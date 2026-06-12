---
name: implement-next-feature
description: Implement the next uncompleted feature (phase) from an implementation plan on its own feature branch — all AFK tasks autonomously, human-in-the-loop tasks via talk-it-through and manual verification, ending in a PR after user approval. Use when invoked with a plan file path, or when the user asks to build/implement the next feature from a plan.
---

# Implement Next Feature

Implement the **next uncompleted feature** of the plan in full on its own feature branch, then
open a PR once the user approves. The plan is the source of truth for what to build and what is
already done.

## Argument

A single argument: the path to the plan file (e.g. `plans/job-tracker-plan.md`). If no argument
is given, ask the user which plan to use before doing anything else.

## Workflow

1. **Read the plan in full.** Read the entire plan file end to end — workflow ground rules,
   architectural decisions, every phase, and the implementation log. Do not skim.

2. **Establish what already exists.** Read the plan's implementation log, then *verify against
   reality* — check git history and the actual files on disk. The log can be stale or wrong;
   trust the code over the log. Note any drift.

3. **Pick the next feature.** The next feature is the first phase with any unchecked tasks
   (`- [ ]`). State clearly which feature you are implementing before you start.

4. **Create or check out the feature branch.** The branch name comes from the phase's
   `**Branch**` field. If the branch already exists (locally or on the remote), check it out and
   continue where work left off. If it doesn't, update `main` and create the branch from it.
   All work for this feature happens on this branch.

5. **Resolve `[decision]` tasks via talk-it-through.** Before writing code that a decision affects,
   resolve every open `[decision]` task by invoking the **talk-it-through** skill and reaching
   mutual agreement. Since this needs the user, **send a push notification first** (see
   notification rule below). Record each agreed decision in the plan.

6. **Implement all AFK tasks.** Build every AFK task in the feature, autonomously, without
   pausing for input. Follow every ground rule the plan states. Stay scoped to this one
   feature — don't pull work forward from later phases. Bias hard toward AFK: if something
   *can* be verified with a test, script, or automated check, do that instead of asking the
   user.

7. **Verify your own work.** Run the relevant tests/checks for everything you built and confirm
   they pass. Report failures honestly with output; never bend tests to pass.

8. **Request manual `[verify]` checks.** Once all AFK work is done and green, present every
   `[verify]` task to the user in one batch — what to check and how. Send a push notification,
   then wait. If verification fails, fix and re-request.

9. **Update the plan.** Check off all completed tasks (`- [x]`), record what was built, key file
   paths, and every decision made via talk-it-through. Leave the plan an accurate source of truth.

10. **Get approval, then open the PR.** When everything is finished — AFK tasks done, tests
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
- Batch human input where possible (one notification per pause, not one per item).
- Verify what's done against the code, not just the log.
- Update the plan before finishing; PR only via createpr, only after approval.
