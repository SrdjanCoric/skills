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

- **hash-it-out**: Interviews you about a plan, design, or idea, one question at a time, until you've reached shared understanding. Walks down each branch of the decision tree and saves the final decisions to a file.
- **write-well**: Writes and revises prose in a direct, human voice. Enforces a strict checklist for stripping the tells that make text read as machine-generated.
