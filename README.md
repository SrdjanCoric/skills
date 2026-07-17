# Skills

My personal [Claude Code](https://claude.com/claude-code) skills, also usable in Codex. Each skill is a folder under `skills/` with a `SKILL.md` that tells the agent when to activate and what to do.

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

The main software workflow is `talk-it-through`, an optional PRD, `to-plan`, `implement-next-task`, and `sync-main`. Implementation includes task review, README updates when needed, final behavior proof, PR creation, and a wait for CI. `sync-main` merges the verified PR and closes the local task. Each skill also works on its own.

- **talk-it-through**: Interviews you about a plan, design, or idea until the decisions are clear. For software work, it inspects the codebase and applies the relevant repository guidelines before saving the agreed decisions.
- **write-a-prd**: Turns the current conversation or a decision document into a PRD and saves it to a local file.
- **to-plan**: Turns a PRD, decision document, or conversation into the smallest independently verifiable vertical tasks under `plans/tasks/`. It checks relevant repository guidelines, records direct dependencies, presents the proposed breakdown, and waits for approval before writing files.
- **implement-next-task**: Claims one eligible task and implements it on its feature branch. It uses TDD for behavior changes, resolves real uncertainty through `talk-it-through`, applies repository guidelines, runs autonomous review and remediation, updates the README when needed, proves the final behavior, and opens a CI-green PR after approval.
- **task-review**: Reviews a branch against repository standards, the task or spec, correctness, and relevant security risks. It automatically fixes current-diff non-security findings and reruns the review for up to two remediation passes. Every security finding requires a plain-English user decision. Results stay in context instead of being written to a review file.
- **software-repository-guidelines**: Applies a shared repository checklist during scoping, planning, implementation, review, or a full final assessment. It loads only the relevant references for ordinary work and requires repository or CI evidence before calling an item complete.
- **create-pr**: Commits the current unit of work, pushes the branch, opens or updates its PR, and waits for CI. It invokes `diagnose` for in-scope CI failures and stops when the current head is ready to merge.
- **sync-main**: Treats an explicit invocation as approval to merge a verified, CI-green PR. It synchronizes local `main`, removes branches whose merged state is proven, changes matching task pointers from `[>]` to `[x]`, and moves their task files to `plans/tasks/done/`.
- **write-well**: Writes and revises prose in a direct, human voice. It includes a separate adversarial audit that removes common machine-written tells.
- **tdd**: Builds features and fixes bugs through observed red-green-refactor cycles. It favors behavior through public interfaces and applies the testing section of the repository guidelines.
- **handoff**: Captures one focused slice of the current work into a handoff document that a fresh agent can continue without re-deriving the context.
- **diagnose**: Works hard bugs and performance regressions through a disciplined loop: build a feedback loop, reproduce, minimize, hypothesize, instrument, fix, and add a regression test.
- **teach**: Turns the current directory into a persistent teaching workspace and teaches one topic over many sessions, one self-contained HTML lesson at a time. Each lesson can collect highlighted questions that shape the next lesson.

## Dependencies

Some skills invoke others, shown below with `→`. The installer does not follow these links, so `--skill` installs only the named skill.

```
talk-it-through    → software-repository-guidelines
to-plan            → software-repository-guidelines
implement-next-task → software-repository-guidelines, tdd, talk-it-through,
                      task-review, write-well, create-pr
task-review        → software-repository-guidelines, write-well, tdd when remediation needs it
create-pr          → write-well, diagnose
tdd                → software-repository-guidelines
write-a-prd        → write-well
teach              → talk-it-through, write-well
diagnose           → handoff
```

`task-review` also uses the environment's code-review capability. It runs a security lens only when the diff touches a relevant trust boundary. In Claude Code it uses `/security-review`; in Codex it uses `$codex-security:security-diff-scan`, which requires the Codex Security plugin.

To install `implement-next-task` and its full transitive dependency chain:

```bash
npx skills@latest add SrdjanCoric/skills \
  --skill implement-next-task \
  --skill software-repository-guidelines \
  --skill tdd \
  --skill talk-it-through \
  --skill task-review \
  --skill write-well \
  --skill create-pr \
  --skill diagnose \
  --skill handoff
```

Run `npx skills@latest add SrdjanCoric/skills` without `--skill` to choose from the full list.

The lifecycle states are `[ ]` ready, `[~]` in progress, `[>]` complete with a CI-green PR awaiting merge, and `[x]` merged into `main`. Only `sync-main` changes `[>]` to `[x]`.
