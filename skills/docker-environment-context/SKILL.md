---
name: docker-environment-context
description: Select, activate, and preserve the correct prod, dev, or load-test Docker Compose context for this Symfony workspace and render environment-aware Docker, Symfony console, database, cache, restart, and service commands. Use when the user asks how to run, build, restart, inspect, or execute commands in project containers; mentions dev, development, prod, production, or load testing; or provides a Docker Compose command whose environment context matters.
---

# Docker Environment Context

## Purpose

Keep every generated command aligned with the environment discussed by the user. Treat production as the project default and development and load testing as explicit Compose overrides.

## Determine the Environment

Resolve the environment in this order:

1. Use an explicit environment in the user's current request or active task context.
   - Map `dev`, `development`, and `для дева` to `dev`.
   - Map `prod`, `production`, `прод`, and `бой` to `prod`.
2. Map `load-test`, `load_test`, and `нагрузочное тестирование` to `load-test`.
3. If the user provides Compose flags, treat `--env-file .env.dev` or `-f docker-compose.dev.yml` as `dev`, and `--env-file .env.load_test` or `-f docker-compose.load-test.yml` as `load-test`.
4. Otherwise use `prod`, because the root `.env` sets `COMPOSE_FILE=docker-compose.yml` and `COMPOSE_PROJECT_NAME=symfony2026`.

Preserve the resolved environment throughout the current task. Do not switch environments unless the user explicitly changes it.

Do not infer the environment from container names or running-container state. The environments use the same Compose service names.

## Activate the Environment

Activate only the Compose control and selector variables in the current shell before rendering a sequence of Docker, npm, or project script commands. Do not source `.env` or `.env.local` into a long-lived interactive shell. Source `.env.compose` so Docker Compose receives `COMPOSE_ENV_FILES` before it resolves interpolation files, then source only the environment-specific selector when needed.

Production:

```bash
set -a
. ./.env.compose
set +a
unset COMPOSE_PROJECT_NAME COMPOSE_FILE
```

Development:

```bash
set -a
. ./.env.compose
. ./.env.dev
set +a
```

Load testing:

```bash
set -a
. ./.env.compose
. ./.env.load_test
set +a
```

`set +a` stops automatic export for future assignments; it does not unset the Compose variables that were just loaded. The selected context therefore remains active for subsequent commands in the same terminal. Repeat activation in every new shell, always source `.env.compose`, and source only the desired selector override when switching contexts. For production, unset any inherited override selectors so Compose falls back to the values in the root `.env`.

This separation intentionally keeps values from `.env.local`, including secrets, out of the interactive shell environment. Only Compose receives them for interpolation, and only values explicitly wired through a service's `environment` or `env_file` configuration enter that container. Host-side scripts that need non-secret project configuration must read `.env` in their own short-lived process; they must not source `.env.local` globally.

Prefer `npm run set:prod`, `npm run set:dev`, or `npm run set:load-test` when the user wants a dedicated interactive shell. These helpers execute that shell as the npm process, so the selected Compose variables remain available inside it without modifying the parent shell. A one-shot `docker compose --env-file ...` command remains valid when the user explicitly wants it, but it does not activate the context for later npm or Bash scripts.

The override files set distinct `COMPOSE_PROJECT_NAME` and `COMPOSE_FILE` values. This separates the Compose projects, generated networks, and named volumes. It does not separate explicitly configured `container_name` values or published host ports, so stop the current stack before switching environments when those resources conflict.

## Render Commands

After activation, use the ordinary Compose prefix for every environment:

```bash
docker compose
```

Write user-facing commands relative to the repository root. Do not add `cd app`.

State the selected environment immediately before the command when it is not already obvious from the user's wording. Output only the selected environment's command unless the user explicitly asks for both variants.

Prefer existing environment-neutral npm scripts when they exactly match the requested operation. Do not add redundant `-f` or `--env-file` flags after activation because the exported `COMPOSE_FILE` supplies the complete Compose file list and `COMPOSE_PROJECT_NAME` selects the project.

## Safety and Ambiguity

- Do not run `docker compose up`, `docker compose build`, `docker compose pull`, Composer installation, or dependency installation. Print the exact command for the user to run.
- Ask which environment to use before a destructive or behavior-sensitive operation only when the current task conflicts with the default or contains contradictory environment signals.
- Use prod without asking when no environment signal exists and the operation is ordinary, because prod is the documented project default.
- Do not output parallel prod, dev, and load-test alternatives merely because they exist.

## Project Examples

Activate production and start its stack:

```bash
set -a
. ./.env.compose
set +a
unset COMPOSE_PROJECT_NAME COMPOSE_FILE
docker compose up -d
```

Activate development and start its stack:

```bash
set -a
. ./.env.compose
. ./.env.dev
set +a
docker compose up -d
```

Activate load testing and run database fixtures:

```bash
set -a
. ./.env.compose
. ./.env.load_test
set +a
npm run db:fixtures
```

Clear the main Symfony cache in the already activated context:

```bash
docker compose exec symfony-cli php -d memory_limit=1G bin/console cache:clear
```

Restart web services in the already activated context:

```bash
docker compose restart symfony-web catalog-web catalog-grpc cart-web api-gateway
```

Run database migrations in the already activated context:

```bash
npm run db:migrate
```

## Load Testing Exception

Commands for k6 use their dedicated Compose file and are not rewritten with the application dev override:

```bash
docker compose -f load-testing/docker-compose.yml run --rm k6 run /scripts/scenarios/checkout.js
```

If the load test targets a dev or prod application endpoint, preserve that target through the load-testing environment variables or script options instead of adding `docker-compose.dev.yml` to the k6 Compose command.

## Configuration Changes

When the request changes Docker Compose files, service environment variables, gateway wiring, or operational scripts rather than only generating a command, also apply the `service-maintenance` skill.
