# `handoff` — design decisions

Redesign of the global `handoff` skill, from a talk-it-through on 2026-06-16. The old skill was a
14-line "summarise the conversation into an `mktemp` file" — it didn't give the next agent enough
context to actually continue. This is the contract the rewrite is built against.

## Problem with the old skill

- Output went to a random `mktemp -t handoff-XXXXXX.md` — unguessable, outside the repo, OS may
  garbage-collect it. A handoff's whole job is to be *found*; this buried it.
- Framed as "summarise the conversation" → transcript recap heavy on what-we-discussed, thin on
  what-to-do-next. The next agent got implementation chatter, not an actionable context package.
- No template, no required structure → quality was luck-of-the-draw.
- Wrote from memory, no verification against actual git/disk state.

## Core principle

> A handoff is built from **two inputs**: the **argument** (what this is about — it scopes the doc
> and selects the template) and the **current conversation** (the source it distills). Extract from
> the conversation only what's relevant to the argument; verify state claims against git and disk;
> never dump the whole transcript.

The argument is the **scope**; the conversation is the **source material**. The doc is the
intersection — a focused context package, not a summary.

## Decisions

### Argument
- **Required.** It names the subject of the handoff.
- **No argument → stop and ask the user** what the handoff should be about. Never silently fall
  back to summarising the whole conversation.

### Three intents → three templates
The argument maps to one of three intents:

- **bug** — "go *fix* this known problem."
- **feature-pivot** — "go *explore/build* this *new* thing we just discovered." Forward-looking:
  a new idea/feature surfaced in the current work, spun off to a fresh agent. **Not** parking and
  resuming existing work.
- **general** — anything else (continue/transfer some other context).

Implemented as files in the skill folder:

```
~/.claude/skills/handoff/
  SKILL.md          # orchestrator: require arg, pick template, distill, write, print path
  bug-handoff.md    # bug template
  feature-pivot.md  # new-exploration template
  # general fallback lives inline in SKILL.md (no file)
```

### Template selection
- **Infer from the argument + conversation, announce the choice, allow override.** e.g. "Using the
  bug-handoff template." Only ask the user when the argument is genuinely ambiguous.
- If it matches neither bug nor feature-pivot → **general fallback**.

### Output location
- Path: **`<repo-root>/handoffs/<branch>/<slug>.md`** — branch and slug both identify it. Branch
  slashes become real folders (`handoffs/feature/synthetic-interview-evals/transcript-reset-bug.md`),
  grouping a branch's handoffs together. `<slug>` derives from the argument.
- **Gitignored.** The skill ensures the project `.gitignore` contains `handoffs/` (append if
  missing). Ephemeral, same-machine continuation scaffolding — not repo history.
- **Overwrite** if the same branch+slug is handed off again. No timestamps.
- The skill ends by **printing the path + a ready-to-paste resume line** for the next session, so
  pickup is one copy-paste, not a hunt.

### Voice & length
- **No `write-well` pass.** A handoff is a precise technical working doc for an agent — value is in
  `path:line` precision and "why this failed," not literary voice.
- **Terse, anti-slop, complete-but-scannable** — roughly a screen or two; longer only when the
  bug/idea genuinely demands it. No filler, no restating the obvious.

### Cross-cutting rules (all templates)
- **Verify state against git/disk, not memory** — "done so far" / "files changed" claims get
  checked.
- **Reference, don't duplicate** — plans, ADRs, PRDs, issues, commits, diffs linked by path/URL,
  never copied (kept from the old skill).

## Template section sets

### `bug-handoff.md` — next agent must *solve a problem*
- **Objective** — the bug to fix (one line, from the argument).
- **Symptom** — what's observably wrong, with trigger/repro if known.
- **What we've tried & ruled out** — each attempt + *why it failed* (so it isn't repeated).
- **Current hypothesis** — leading theory on the cause, if any.
- **Relevant files & locations** — `path:line` + one-line role each.
- **Where it stands** — verified vs git/disk: what's changed so far, what's untouched.
- **Next steps** — the concrete next move.
- **Gotchas** — non-obvious knowledge not in any artifact.
- **Suggested skill** — `diagnose`.

### `feature-pivot.md` — next agent must *start a newly-identified piece of work*
- **Objective** — the new feature/direction (from the argument).
- **What sparked it** — the insight/moment in the current work that led here (load-bearing: the new
  agent has none of this conversation).
- **What we've figured out so far** — shape, scope, constraints, decisions already reached.
- **How it relates to existing code** — what it builds on / touches / replaces, `path:line` + role.
- **Open questions** — what still needs deciding.
- **Suggested starting point** — usually *not* "write code" but "flesh it out."
- **Gotchas / constraints**.
- **Suggested skill** — `talk-it-through`.

### General fallback (inline in `SKILL.md`)
Universal spine: **Objective · Context · What's been done/tried · Relevant files & locations ·
Where it stands · Next steps · Gotchas**. **Suggested skill** — `talk-it-through`.

## Suggested-skills principle
Point at the **single natural next step, not the whole downstream pipeline.** The handoff doesn't
presume the route (`write-a-prd` → `prd-to-plan` → `implement-next-feature`); it names the one
obvious next move and the user invokes the rest if warranted.

- bug → `diagnose`
- feature-pivot → `talk-it-through`
- general → `talk-it-through`
