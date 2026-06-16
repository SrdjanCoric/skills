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

The first five work as a pipeline: talk through an idea, turn the decisions into a PRD, break the PRD into a phased plan, implement it one feature at a time, then review each finished feature. Each also works on its own.

- **talk-it-through**: Interviews you about a plan, design, or idea, one question at a time, until you've reached shared understanding. Walks down each branch of the decision tree and saves the final decisions to a file.
- **write-a-prd**: Turns the current conversation or a decision document into a PRD and saves it to a local file.
- **prd-to-plan**: Breaks a PRD into a multi-phase implementation plan built from tracer-bullet vertical slices, saved to `./plans/`.
- **implement-next-feature**: Implements the next uncompleted feature from a plan on its own branch. Runs the autonomous tasks on its own, pulls you in for decisions and manual verification, and ends in a PR after your approval.
- **feature-review**: Reviews a finished feature branch with a panel of parallel agents (standards, spec faithfulness, bugs, and security) and synthesizes their findings into one readable review file under `reviews/`.
- **write-well**: Writes and revises prose in a direct, human voice. Enforces a strict checklist for stripping the tells that make text read as machine-generated.
- **sync-main**: Checks out the main branch and pulls the latest changes, warning you first if uncommitted work would be overwritten.
- **create-pr**: Commits and pushes the current changes if needed, then opens a PR with a structured, auto-generated description (overview plus per-file key changes).
- **tdd**: Builds features and fixes bugs test-first with a red-green-refactor loop. Ships with reference notes on writing tests, mocking, refactoring, and module design.
