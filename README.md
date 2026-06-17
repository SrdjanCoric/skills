# Skills

My personal [Claude Code](https://claude.com/claude-code) skills. Each skill is a folder under `skills/` with a `SKILL.md` that tells Claude when to activate and what to do.

## Installation

```bash
npx skills@latest add SrdjanCoric/skills
```

The installer lists every skill in the repo and lets you pick which ones to install. Run it again later to pull updates.

To install a single skill directly:

```bash
npx skills@latest add SrdjanCoric/skills --skill write-well
```

## Skills

The first five work as a pipeline: talk through an idea, turn the decisions into a PRD, break it into self-contained tasks under a master plan, implement the next task on its own branch, then review the finished task. Each also works on its own.

- **talk-it-through**: Interviews you about a plan, design, or idea, one question at a time, until you've reached shared understanding. Walks down each branch of the decision tree and saves the final decisions to a file.
- **write-a-prd**: Turns the current conversation or a decision document into a PRD and saves it to a local file.
- **to-plan**: Turns a PRD, decision doc, or the current conversation into self-contained task files under `plans/tasks/`, each appended as a pointer to the project's single master plan.
- **implement-next-task**: Implements the next uncompleted task from the master plan on its own branch. Runs the autonomous work on its own, pulls you in for decisions and manual verification, and ends in a PR after your approval.
- **task-review**: Reviews a finished task branch with a panel of parallel agents (standards, spec faithfulness, bugs, and security) and synthesizes their findings into one readable review file under `reviews/`.
- **write-well**: Writes and revises prose in a direct, human voice. Enforces a strict checklist for stripping the tells that make text read as machine-generated.
- **sync-main**: Checks out the main branch and pulls the latest changes, warning you first if uncommitted work would be overwritten.
- **create-pr**: Commits and pushes the current changes if needed, then opens a PR with a structured, auto-generated description (overview plus per-file key changes).
- **tdd**: Builds features and fixes bugs test-first with a red-green-refactor loop. Ships with reference notes on writing tests, mocking, refactoring, and module design.
- **handoff**: Captures a focused slice of the current work (a bug, a new task, some context) into a handoff document a fresh agent can pick up without re-deriving everything.
- **diagnose**: Works hard bugs and performance regressions through a disciplined loop: build a feedback loop, reproduce, minimize, hypothesize, instrument, fix, then add a regression test.
