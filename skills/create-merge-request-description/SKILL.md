---
name: create-merge-request-description
description: Generate project-style Markdown merge request descriptions for this Symfony learning repository. Use when the user asks to draft, output, prepare, or format an MR/PR description from an existing task file, task number, pull request URL, branch, or recently completed project work.
---

# Create Merge Request Description

## Overview

Generate a ready-to-paste Markdown merge request description. Prefer outputting Markdown directly in the chat instead of editing project files, unless the user explicitly asks to save it.

Use the existing project task document and current branch context as the source of truth. Keep the result concise, factual, and aligned with the task scope.

## Workflow

1. Check `.aiignore` before reading project files.
2. Identify the task number from the user request, pull request title/context, or nearby conversation.
3. Read the matching task file, usually `symfony/docs/task-N.md`, `symfony/docs/task-N.M.md`, or `symfony/docs/task-N-M.md`.
4. Get the current branch with `git rev-parse --abbrev-ref HEAD` when GitHub blob links are needed.
5. Generate only the MR description Markdown unless the user asks for explanation too.

## Output Template

Use this structure by default:

```md
## Summary

Backend Merge Request N: <backend MR URL or TBD>
Frontend Merge Request N: <frontend MR URL or TBD>
Task file: [symfony/docs/task-N.md](https://github.com/ivanserg0692/symfony2026/blob/<branch>/symfony/docs/task-N.md)

<One or two short paragraphs explaining what the MR implements and why.>

## Scope

- <Main implemented or planned change.>
- <Important boundary or integration point.>
- <Relevant documentation/API/architecture impact.>

## <Optional Domain Section>

- <Use focused sections such as Target Endpoints, Architecture Notes, Public API Boundary, Security Notes, or Testing Notes only when the task calls for them.>

## Out Of Scope

- <Explicit non-goals from the task file or conversation.>

## Task

Task file: `symfony/docs/task-N.md`
```

## Link Rules

- Use `Backend Merge Request N: <url>` when the user provides a PR/MR URL.
- Use `Frontend Merge Request N: TBD` unless a frontend MR URL is known.
- Use repository blob links in this form:
  `https://github.com/ivanserg0692/symfony2026/blob/<branch>/<path>`
- Include MR result links only when matching `symfony/docs/mr-task-*` files already exist or the user asks to reference them.
- Include additional docs or diagram links only when they are directly relevant to the task and exist in the repository.

## Content Rules

- Base the Summary and Scope on the task file and what the user says was done.
- Do not claim implementation details that are not present in the task file, local diff, or conversation.
- Clearly separate internal endpoints from public API boundaries when the task involves API Gateway or service contracts.
- Use English for the MR description unless the user asks for Russian or bilingual output.
- Keep bullets concrete and action-oriented.
- Use `TBD` for unknown links instead of inventing URLs.

## No File Edits By Default

This skill normally returns Markdown only. Follow project documentation approval rules before editing README files, MR log files, or task documents.
