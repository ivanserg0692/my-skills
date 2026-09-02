---
name: documentation-toc
description: Add or maintain generated table-of-contents blocks and README discoverability for Markdown documentation. Use when creating or expanding project docs, README files, merge request notes, task docs, contract docs, or testing docs that need navigable generated contents via doctoc/project scripts, or when a tracked/staged target documentation file may be disconnected from the main README navigation.
---

# Documentation TOC

## Overview

Use this skill when a Markdown documentation file becomes large enough that navigation would help. Prefer generated TOC blocks over handwritten contents so links stay in sync with headings.

## When To Add A TOC

Add a generated TOC when a Markdown file has several meaningful sections, multiple language sections, long task/MR history, API or contract documentation, or repeated subsections that make scanning difficult.

Do not add a TOC to very small files where it adds noise. When unsure, add it if the file already has a document title plus several second- or third-level headings.

## Marker Pattern

Place the marker block immediately after the document title and before the first content section:

```md
# Document Title

<!-- START doctoc -->
<!-- END doctoc -->

## English
...
```

Do not hand-edit generated TOC content between these comments. Re-run the project TOC generator instead.

## Multilingual Documents

Preserve the project Markdown nesting convention:

- use `#` only for the document title;
- use `##` for top-level language sections such as `English` and `Русский`;
- use `###` and deeper headings inside each language section.

The generated TOC should reflect this nesting instead of flattening localized sections.

## Project Workflow

1. Check `.aiignore` before reading or editing project files.
2. For documentation changes, show the planned documentation changes and wait for the user to confirm with `делаем`.
3. Add the doctoc marker block after the `#` title when the file needs generated navigation.
4. Before adding a Markdown file to the shared project TOC script, check its git state. Add tracked files and newly staged/indexed files. Do not add untracked files that are not staged; treat them as temporary working documentation unless the user explicitly says they are part of the project documentation.
5. Ensure the project TOC script includes only recognized documentation files that should be maintained by the shared generator. If it does not, update the script rather than running an ad hoc one-off command.
6. Check README discoverability for the target documentation files being created or edited. Run this check only when the target file itself is tracked by git or staged/indexed. A navigation path is valid only when every intermediate Markdown file in the path is also tracked or staged/indexed. If a path exists only through an untracked/unstaged intermediate file, warn the user and do not blindly add a direct `README.md` link.
7. Run the project generator from the repository root:

```bash
npm run readme:toc
```

8. Verify the diff keeps generated README/API sections untouched except for generated TOC output.

## README Discoverability

When creating or substantially editing Markdown documentation, decide whether it is discoverable enough from the main `README.md`.

Only perform this check for target project documentation files being created or edited when the target file itself is recognized by git:

- files already tracked by git;
- new files already staged/indexed for inclusion.

Do not perform this check for untracked and unstaged Markdown files. Treat them as temporary scratch, context, summary, or agent working files unless the user explicitly promotes them.

A file is sufficiently discoverable when the main `README.md` links to it directly, or when the main `README.md` links to a recognized documentation entrypoint that links to it. Recognized entrypoints include dedicated documentation overviews, contract README files such as `grpc-contracts/README.md`, testing documentation such as `TESTING.md`, or other stable docs index files.

The full path from the main `README.md` to the target file must go through tracked or staged/indexed Markdown files. If an intermediate file in that path is untracked and unstaged, treat the path as unstable: warn the user that the target is reachable only through a temporary file, but do not add a direct link to the main `README.md` without a separate explicit confirmation.

Do not count task files or merge request logs as documentation entrypoints for this purpose. Files such as `symfony/docs/task-*.md` and `symfony/docs/mr-task-*.md` are planning/history artifacts; a link that exists only through them does not make current project documentation discoverable enough.

If an eligible target documentation file is important but disconnected, surface that as a recommendation rather than making the change immediately. Explain the missing or unstable path briefly, name the proposed README link target if one is appropriate, and ask for a separate confirmation before editing `README.md`.

## Script Convention

Use the existing `readme:toc` npm script as the source of truth for generated Markdown contents. Extend it only with recognized documentation files: files already tracked by git, or new files that are staged/indexed for inclusion. Do not extend it with untracked scratch, context, temporary summary, or agent working files unless the user explicitly promotes them to project documentation. When adding another documentation file to generated TOC management, extend that script with the file path, for example:

```json
"readme:toc": "doctoc README.md grpc-contracts/README.md"
```

Keep user-facing paths relative to the repository root.
