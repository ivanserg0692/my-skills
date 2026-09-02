---
name: maintain-skills-index
description: Maintain the local project skills index and its internal, adapted, and external origin groups whenever Codex local skills are added, renamed, removed, recategorized, adapted, or materially updated under .codex/skills.
metadata:
  short-description: Keep local skills index current
---

# Maintain Skills Index

Use this skill when work changes local project skills under `.codex/skills/`, including creating, renaming, deleting, recategorizing, or materially updating a skill.

## Required Behavior

- Check `.aiignore` before reading or modifying project files.
- Keep every local skill as a direct child of `.codex/skills/` with this structure:
  `.codex/skills/<skill-name>/SKILL.md`.
- Do not move skills into category subdirectories; Codex discovery depends on direct child folders.
- Update `.codex/skills/INDEX.md` whenever a local skill is added, renamed, removed, or recategorized.
- If a skill's purpose changes enough that its category is no longer accurate, update `.codex/skills/INDEX.md`.
- Classify every indexed skill by origin:
  - **Internal Skills:** written by the project owner.
  - **Adapted Skills:** obtained from an external source and substantially adapted for the project owner's needs.
  - **External Skills:** downloaded and used without substantial adaptation.
- Base origin classification on where the skill came from, not only on its current contents. An externally sourced skill may move from `External Skills` to `Adapted Skills` after substantial adaptation, but it must not become an internal skill.
- Record the original source for adapted and external skills when it is known.
- List only local project skills in `.codex/skills/INDEX.md`; do not list system skills or plugin skills.

## Workflow

1. Read `.codex/skills/INDEX.md`.
2. Inspect local skill folders with `.codex/skills/*/SKILL.md`.
3. Apply the requested skill change.
4. Update `.codex/skills/INDEX.md` in the same turn, preserving the skill's origin group and thematic subgroup.
5. Validate any created or edited skill with `quick_validate.py` when practical.

## User Communication

- When a requested skill change requires an index update, state that `.codex/skills/INDEX.md` is being updated as part of the change.
- If the user asks to group skills by folders, explain that categories belong in `INDEX.md`, while physical skill folders must stay direct children of `.codex/skills/`.
