# `task-review` — design decisions

Design summary from the talk-it-through on 2026-06-16. This is the contract the
`task-review` skill (and the `implement-next-task` integration) is built against.

## Goal

A multi-lens code-change reviewer that runs **fully AFK at the end of feature implementation**
and, on demand, **standalone**. It fans out to several review lenses — each in its own agent —
and synthesises the findings into **one human-readable markdown file** that a person reads
during the manual-verification pause.

It supersedes the original two-axis `review` skill. Both skills are **global**
(`~/.claude/skills/`), not project-local.

## Decisions

### Scope & identity
- **Not a correctness reviewer itself.** Bug/security/efficiency hunting is delegated to the
  existing `/code-review` and `/security-review` skills, which join the panel as separate agents.
  This skill's native value is the two lenses those don't cover: **Standards** and **Spec**.
- **Four lenses, not two:**
  - **Standards** — does the diff follow this repo's *documented* coding standards? Includes a
    **test-quality** check (are the tests meaningful, do they cover the spec's named edges, are
    they bent/tautological/over-mocked?) — justified because "never bend tests" and the `tdd`
    skill's good-vs-bad-test rules are documented standards. `/code-review` hunts bugs but does
    **not** audit test meaningfulness, so this is non-redundant.
  - **Spec** — does the diff faithfully implement the originating spec? Missing/partial
    requirements, scope creep, mis-implementations.
  - **Bug** — delegated to `/code-review`.
  - **Security** — delegated to `/security-review`, **conditional** (see below).
- **Why correctness is still reviewed despite TDD:** TDD only covers behaviours the author chose
  to test (`tdd` skill: "you can't test everything"), tests through public interfaces (misses
  security/concurrency/edge cases), and can't audit itself. So a review pass is still warranted —
  concentrated in test-quality (Standards lens) and bug-hunting (`/code-review`).

### Execution model
- **One agent per lens**, run in parallel, so contexts don't pollute each other.
- **Hybrid execution (c):** Standards + Spec run from **inline briefs** defined in this skill
  (they have no external home). Bug + Security **invoke the real `/code-review` and
  `/security-review`** so their logic is never forked/drifted.
- **Invocation feasibility — VERIFIED 2026-06-16.** A `general-purpose` subagent **can** invoke
  `/code-review` via the Skill tool. Key learning: `/code-review` is a **prompt skill**, not a
  self-contained tool — invoking it *injects a playbook* the calling agent then executes inline;
  it does **not** spawn a sub-process that returns a finished report. Its literal playbook wants to
  fan out its *own* nested finder/verifier agents, but a **leaf subagent cannot spawn sub-agents**,
  so the Bug/Security lens executes the playbook **inline in a single agent** (several tool
  round-trips) and loses `/code-review`'s internal parallelism. That's acceptable at default
  effort: we get the real skill's **checklist + output contract** without its multi-agent depth.
  No network/PR access needed (`--comment`/`--fix`/`ultra` are opt-in). The lens agent must be told
  to (a) execute the playbook inline, not try to nest agents, and (b) honour the JSON Finding
  contract — `/code-review` returns prose otherwise.
- **Bug-lens effort: default** (not `high`) — keeps the auto-run cheap.
- **Lens agents return structured findings, not prose.** A single **synthesis author** then
  writes the whole report **through the `write-well` skill** — one voice, human-readable, not four
  stitched-together sections.

### Finding schema (every lens, same shape)
- `axis` — `standards` | `spec` | `bug` | `security`
- `severity` — `blocker` | `major` | `minor` | `nit`
  - `blocker` = must fix before merge; `major` = hard standards violation / partial spec /
    should-fix; `minor` = judgment call; `nit` = cosmetic.
  - Maps the original "hard violation vs judgment call" onto major vs minor.
- `location` — `file:line` (or `file` + hunk)
- `claim` — one sentence: what's wrong
- `evidence` — **mandatory citation**: Standards → the documented rule + where it's written;
  Spec → the quoted spec/plan line; Bug/Security → the concrete failing scenario. No finding
  without evidence.
- `suggestion` — optional one-line fix direction

### Synthesis rules
- Keep the lenses as **distinct labeled sections** (Standards / Spec / Bug / Security). Rank by
  severity **within** a section, **never across** — preserves the original "no axis masks another."
- **Dedupe allowed** (merge identical findings across lenses, note which lenses flagged it).
- **No silent drops** — if the author demotes/merges a finding it says so; it never disappears.
- Lead with a short human summary: counts per lens + the single worst finding.

### Inputs (AFK vs standalone)
- **AFK (invoked by `implement-next-task`):** caller injects `base` and `spec`.
  - `base` = `main` (feature branch is cut from main), diff via `git diff main...HEAD` (three-dot).
  - `spec` = the **verbatim active plan phase + referenced decision docs** (the real contract the
    code was built against — better than a GitHub issue).
  - No questions asked.
- **Standalone (`/task-review`):** self-resolves — ask for `base` if not given; hunt for the
  spec in order (commit issue refs → path arg → PRD under `docs/`/`specs/`/`plans/` → ask). If no
  spec exists, the Spec lens reports "no spec available."

### Security gating (AFK-safe)
- A **path/content heuristic** in the orchestrator decides whether to add the security agent:
  signals include auth/session, secret/token/key handling, input parsing/deserialization,
  SQL/shell/subprocess, network/external I/O, prompt/guardrail code, file uploads, crypto.
- The signal list lives **in the skill**, visible and tunable.
- When security is **skipped**, the report says so explicitly ("security skipped — no
  security-relevant surface touched") so a silent skip is never mistaken for "security passed."

### Output file
- Path: **`<repo-root>/reviews/<featureName>-review.md`** (repo root, sibling to `plans/`).
- `featureName` = branch name with a leading `feature/` stripped
  (`feature/synthetic-interview-evals` → `synthetic-interview-evals`). Standalone with no plan:
  current branch name, same stripping.
- **Overwrite on re-run** (no timestamp) — the file always reflects the branch's latest state.
- **Gitignored.** Since the skill is global, on each run it **ensures the current project's
  `.gitignore` contains `reviews/`** (appends if missing). It's ephemeral, branch-scoped
  scaffolding; `plans/` is the durable record.

### Integration with `implement-next-task` (global)
Insert a new AFK step **between current step 7 (verify own work / tests green) and step 8 (manual
`[verify]`)**:
- After tests are green, invoke `task-review` with `base=main` and `spec=`the verbatim active
  phase + decision docs.
- It writes `reviews/<featureName>-review.md` fully AFK.
- Step 8's manual pause now covers **both** the `[verify]` checks **and** reading the review doc —
  one notification, one human stop. The review never blocks the run on its own findings.

### Naming & portability
- Skill name: **`task-review`** (avoids collision with the existing `/review`, `/code-review`,
  `/gh-review-pr`; pairs with `implement-next-task`).
- Rewrite the old "two axes" description to reflect four lenses + file output.
- **Drop the hard dependency** on `/setup-matt-pocock-skills` and `docs/agents/issue-tracker.md`;
  the issue tracker is now just one optional Spec fallback, never a precondition.

### Housekeeping note
The original pasted `review` source had ~6–8 char dropouts at intervals ("Pin the fixed
pat…ever", "Anythinhe repo", "standard.e standard", "A change cass one axis") — a copy/transmission
glitch. The clean text is reconstructed in this design; the new SKILL.md is authored fresh, so the
corruption does not carry over.

## Open items for build
- ~~Verify a spawned subagent can invoke `/code-review` / `/security-review`~~ — **DONE**, it can
  (see Execution model). Apply the same expectation to `/security-review` (also prompt-driven).
- ~~Decide the `/code-review` effort level~~ — **default**.
- Constrain the Bug lens to emit `axis: "bug"` for *all* its findings (the test showed
  `/code-review` mapping onto its own categories like `efficiency`/`robustness`); optionally carry
  the original category in a free-text note inside `claim`.
