---
name: create-pr
description: Create a PR from current changes with an auto-generated description. Use when the user wants to create a pull request, open a PR, or mentions "create pr". Takes an optional feature name as an argument.
---

# Create Pull Request

Create a pull request for the current changes. If the user provided a feature name, use it as the feature context; otherwise infer it from the changes.

## Gather current state

Run these first to understand where things stand:

- `git branch --show-current` - current branch
- `git status --short` - working tree status
- `git diff --cached --stat` and `git diff --stat` - staged and unstaged changes
- `git log main..HEAD --oneline` (fall back to `origin/main..HEAD`) - commits not on main
- `git diff main...HEAD` (fall back to `origin/main...HEAD`) - full diff from main

## Task

Follow these steps:

1. **Review the changes**
   - Analyze the diff to understand what was modified
   - Identify the main purpose of the changes
   - Note any breaking changes or dependencies

2. **Stage any unstaged changes** (if appropriate)
   - If there are unstaged changes relevant to this feature, stage them
   - If unstaged changes are unrelated, leave them for a separate PR

3. **Create a commit** (if needed)
   - If there are staged but uncommitted changes, create a commit
   - Use conventional commit format: `type(scope): description`
   - **IMPORTANT**: Never mention Claude Code or any AI assistant in commit messages

4. **Push the branch**
   - Ensure all commits are pushed to origin
   - Create tracking branch if needed

5. **Generate PR content**

   Invoke the `write-well` skill first and write all PR prose (the description and any PR
   comments) following its voice and anti-AI checklist.

   Title format: `type(scope): brief description`

   Description format:
   ```
   # Overview
   [Paragraph explaining what was done at a high level. Can include numbered points for major components/features added. Use **bold** for emphasis on key items.]

   ## Key Changes
   **path/to/file1.ts**
   - [Description of what was changed in this file]

   **path/to/file2.ts**
   - [Description of what was changed in this file]

   **path/to/file3.ts**
   - [Description of what was changed in this file]
   ```

   Guidelines for the description:
   - **Overview**: Write a clear summary of the overall purpose. If multiple components were added, use numbered list with bold labels (e.g., `1. **Feature name** - description`)
   - **Key Changes**: List each modified file with its full path in bold, followed by bullet points describing what was changed in that file. Be specific about new types, functions, configurations, or modifications made.
   - **IMPORTANT**: Never include any mention that the PR was generated with Claude Code or any AI assistant. The PR should appear as a normal human-written PR.

6. **Create the PR**
   - Use `gh pr create` with the generated title and body
   - Set appropriate labels if available
   - Request reviewers if specified in project settings

7. **Output the PR URL**
   - Provide the link to the newly created PR
