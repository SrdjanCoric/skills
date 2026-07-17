---
name: talk-it-through
description: Interview the user about a plan, design, or idea until reaching shared understanding, resolving each branch of the decision tree. Use when the user wants to stress-test a plan, explore a feature or project idea, talk through a topic, or prepare software work for planning. For software repositories, inspect the current system and selectively apply software-repository-guidelines in scope mode.
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.

Match the depth to the decision. For a bounded choice, stay lightweight. For a new application,
workflow, or cross-cutting feature, first classify the work as greenfield or an existing-app
extension, then build enough of this model to plan safely:

- current state and target-state delta,
- product promise, domain nouns, actors, and permissions,
- happy, repeat, failure, cancellation, resume, and cleanup scenarios,
- persisted state, external dependencies, and failure policies,
- UX/communication, safety boundaries, and acceptance proof.

For software work, invoke `software-repository-guidelines` in `scope` mode while building this
model. Load only relevant references and distinguish current requirements from future repository
improvements. Include the references loaded, applicable requirements, and expected proof in the
final summary so `to-plan` can consume them without rediscovery.

Once we reach shared understanding, give a concise in-chat summary of the final decisions and stop.
For substantial software planning, format it as a planner brief covering the model above, explicit
non-goals, unresolved checkpoints, and proof. **Do not** save a decision doc by default — the usual
next step is to invoke the `to-plan` skill to turn the understanding into a task. Write a summary to
`plans/decisions/<feature-name>.md` only if I explicitly ask you to.
