# Bug handoff template

The next agent must **solve a problem**. Fill every section; if one is genuinely empty say so
("no hypothesis yet") rather than dropping it. Keep it terse and `path:line`-precise.

## Objective
One line: the bug to fix (from the argument).

## Symptom
What's observably wrong — error, wrong output, crash. Include the trigger or repro steps if known.

## What we've tried & ruled out
Each attempt and **why it failed**, so the next agent doesn't repeat it. This is the highest-value
section — it's the work that isn't recorded anywhere else.

## Current hypothesis
The leading theory on the cause, if any. State the uncertainty honestly.

## Relevant files & locations
`path:line` + a one-line role for each (e.g. `simulation.py:381 — redirect branch that wipes the
transcript`).

## Where it stands
Verified against git/disk, not memory: what's been changed so far (if anything), what's still
untouched.

## Next steps
The concrete next move to make.

## Gotchas
Non-obvious knowledge not captured in any artifact.

## Suggested skill
`diagnose`.
