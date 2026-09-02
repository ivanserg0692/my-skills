---
name: create-project-task
description: Create or update project task documents in this Symfony learning project's established task format. Use when the user asks to create, draft, describe, formalize, or register a new task, roadmap item, architecture task, implementation task, or planned work item for this repository.
---

# Create Project Task

## Overview

Create task documents using the project's built-in task template. Prefer the template below over re-reading older task files every time.

Use existing task files only when you need to determine the next task number, verify a specific local convention, or the user explicitly asks to mirror an existing task.

## Workflow

1. Check `.aiignore` before reading or writing project files.
2. If creating a project task, place it under `symfony/docs/task-N.md` unless the user requests another path.
3. Determine `N` from the user request. If absent, inspect existing `symfony/docs/task-*.md` filenames and use the next number.
4. Draft the task from the built-in template. Do not require reading previous tasks for normal task creation.
5. Write Russian first, then English.
6. Treat README updates as separate documentation edits. Show the planned README change and wait for explicit confirmation before editing it.
7. When creating MR result files, also plan a README index update that links to the new MR files from the related task entry.
8. Do not manually edit generated README sections or generated API endpoint blocks.
9. If the task changes the API contract, update OpenAPI/Swagger text with the implementation when that implementation work happens.

## Task Template

Use this structure for a normal task:

```md
# Task N

## RU

### Название
...

### Описание задачи
...

### Цель
...

### Критерии приемки
- ...

### Технический подход
- ...

### Как тестировать
- ...

### Примечания
- ...

## EN

### Title
...

### Task Description
...

### Goal
...

### Acceptance Criteria
- ...

### Technical Approach
- ...

### How To Test
- ...

### Notes
- ...
```

## Architecture Tasks

For architecture-heavy tasks, add this section after `Goal` in both languages:

```md
### Архитектурный контекст
...
```

```md
### Architecture Context
...
```

Use it to capture service boundaries, transport choices, lifecycle rules, data ownership, integration contracts, and known tradeoffs.

## README Index Pattern

When the user confirms a README index update, add a concise task entry in the existing Tasks section:

```md
#### `Task N` - planned
- Backend Merge Request N: https://github.com/ivanserg0692/symfony2026/pull/N
- Frontend Merge Request N: https://github.com/ivanserg0692/symfony2026-frontend/pull/N
- Task file: [symfony/docs/task-N.md](symfony/docs/task-N.md)
- MR result (EN): [symfony/docs/mr-task-N-en.md](symfony/docs/mr-task-N-en.md)
- MR result (RU): [symfony/docs/mr-task-N-ru.md](symfony/docs/mr-task-N-ru.md)
```

Use `planned` for future work, `in progress` only when work has started, and `done` only after completion.

If a backend or frontend merge request does not exist yet, use `TBD` for that line instead of inventing a URL.

If MR result files do not exist yet, omit those lines until the files are created.

After manual README index edits, ask the user to run or run when allowed by project rules:

```bash
cd app
npm run readme:toc
```

## MR Result Files

Do not mix task definition and MR result history.

Use separate files when documenting results:

- `symfony/docs/mr-task-N-en.md`
- `symfony/docs/mr-task-N-ru.md`

After creating these files, update the README task index with links to both MR result files, following the README confirmation rule above.

Append chronological updates to MR result files instead of rewriting older history unless correcting stale information.
