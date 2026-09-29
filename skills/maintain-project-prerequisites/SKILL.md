---
name: maintain-project-prerequisites
description: Keep this project's external CLI prerequisite check and README list current when a project workflow adds, changes, or removes a required host-side tool. Use for changes to scripts or workflows that affect Docker, Docker Compose, kubectl, OpenSSL, or other external commands; not for PHP, Composer, or npm package dependencies inside project environments.
metadata:
  short-description: Keep external prerequisites current
---

# Maintain Project Prerequisites

Use this skill when a project workflow starts requiring a new external command, stops requiring one, or changes how an existing command is invoked on the machine running that workflow.

## Source of Truth

- Inspect the actual project script or documented command that uses the tool. Do not add a prerequisite solely because it might be useful later.
- Keep the `prerequisites` list in `scripts/check-prerequisites.mjs` aligned with required host-side tools. `package.json` exposes it as `npm run prerequisites:check`.
- Keep the English and Russian external prerequisites sections in `README.md` aligned with the check. Describe each tool's project purpose without installation commands.
- The repository root is `app` for user-facing commands. Do not include `cd app` in project documentation.

## Check Behavior

- Check the command and subcommand the workflow actually needs; Docker Compose is checked through `docker compose`, not as a separate `docker-compose` executable.
- Report each dependency clearly and exit with a nonzero status if any required command is unavailable or fails its client-side version check.
- Keep the check read-only and cross-platform where practical. It must not install tools, start containers, or require a running Docker daemon or Kubernetes cluster merely to verify CLI availability.
- Do not put project package dependencies or container-internal commands in this host-side checklist unless a user-facing host workflow truly requires them.

## Validation and Documentation

- Follow project approval rules before editing code or documentation. Do not manually edit generated README sections.
- After changing the check, run `node --check scripts/check-prerequisites.mjs` and `npm run prerequisites:check` from the repository root when practical. A missing tool on the current machine is an expected check result, not a reason to install it.
- After changing README headings, regenerate its table of contents with `npm run readme:toc` from the repository root.
- Report which prerequisite changed and whether validation covered only CLI availability or an actual service connection.
