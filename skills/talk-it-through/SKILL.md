---
name: talk-it-through
description: Interview the user relentlessly about a plan, design, or idea until reaching shared understanding, resolving each branch of the decision tree. Use when user wants to stress-test a plan, explore a feature or project idea, wants to talk through a topic, or mentions "talk it through".
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.

Once we reach shared understanding, give a concise in-chat summary of the final decisions and stop. **Do not** save a decision doc by default — the usual next step is to invoke the `to-plan` skill to turn the understanding into a task. Write a summary to `plans/decisions/<feature-name>.md` only if I explicitly ask you to.
