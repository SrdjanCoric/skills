---
name: write-a-prd
description: Turn either the current conversation context or a decision document passed in as an argument into a PRD and save it to a local file. Use when the user wants to create a PRD from the current context or from a decision document.
---

This skill produces a PRD from one of two sources. If a decision document was passed in as an argument, use that document plus the codebase understanding as the source. Otherwise, use the current conversation context plus the codebase understanding. Either way, do NOT interview the user, just synthesize what you already know.

## Argument

An optional argument: the path to a decision document (e.g. a design doc or decisions log). If it's given, that document is the primary source for the PRD. If no argument is given, fall back to the current conversation context.

## Process

1. Explore the repo to understand the current state of the codebase, if you haven't already.

2. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Check with the user that these modules match their expectations. Check with the user which modules they want tests written for, and at what boundary.

3. Use the template below to write the PRD. Write all of its prose through the `write-well` skill so the final output follows that skill's voice and anti-AI checklist. Save the PRD to a local Markdown file.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>