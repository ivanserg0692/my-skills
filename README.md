# Codex Skills

This repository versions the Codex skills used in the Symfony workspace while keeping machine-specific Codex configuration local.

## Repository Contents

- `skills/` contains all installed project skills and their supporting resources.
- `skills/INDEX.md` is the catalog of skills grouped by origin.
- `AGENTS.md.example` is an optional starting point for version-controlled Codex instructions.
- `config.toml` remains local and is intentionally ignored by Git.

## Skill Origin Groups

The skills index uses three origin groups:

- **Internal**: authored locally by the repository owner.
- **Adapted**: obtained externally and then substantially changed for this workspace.
- **External**: downloaded and retained without substantial local adaptation.

An external skill may become adapted after meaningful changes. An externally sourced skill does not become internal.

When the original external source is known, record it in `skills/INDEX.md`. A local archive filename is not an original source by itself.

## AGENTS.md Support

Codex discovers project instructions from `AGENTS.md` at the project root and from applicable nested directories. The active project file for this workspace therefore remains outside this repository at `../AGENTS.md`.

To version those instructions here later:

1. Copy the active instructions into `AGENTS.md`, or rename `AGENTS.md.example` to `AGENTS.md` and expand it.
2. Make `../AGENTS.md` a symbolic link to `.codex/AGENTS.md`, or maintain the active file by an explicit synchronization step.
3. Check the effective instructions in a fresh Codex session after changing the setup.

Do not replace the existing project `AGENTS.md` without reviewing and preserving its current rules.

## Skill Discovery Compatibility

The skills currently live under `.codex/skills` for this workspace. Current Codex documentation describes `.agents/skills` as the standard repository-scoped location and supports symlinked skill directories.

If migration becomes necessary, keep this repository as the canonical source and point `../.agents/skills` to `../.codex/skills`. Verify discovery in a fresh Codex session before removing any compatibility setup.

## Initial Version

Review the files before creating the first commit:

```bash
git status --short
git add .gitignore README.md AGENTS.md.example skills
git commit -m "Initialize versioned Codex skills"
```

