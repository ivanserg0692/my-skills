---
name: service-maintenance
description: Maintain shared configuration and operational scripts when Symfony microservices are added, renamed, exposed through the API Gateway, or connected to shared infrastructure. Use when updating docker-compose services, service env variables, database maintenance targets, internal service URLs, gateway/OpenAPI route generation, or service onboarding rules.
metadata:
  short-description: Keep Symfony service wiring consistent
---

# Service Maintenance

Use this skill when a Symfony service is added, renamed, removed, or connected to shared runtime infrastructure.

## Required Checks

- Check `.aiignore` before reading or modifying project files.
- Inspect existing service patterns before changing shared configuration.
- Treat the repository root as `app` for user-facing project commands.
- Do not run `docker compose up`, `docker compose build`, `docker compose pull`, or Composer install/update/require commands.
- Ask before changing behavior-sensitive authentication, authorization, CSRF, token, retry, cache, or API-client behavior.
- Before moving service-specific env values into the root `.env`, explain the variables and get user approval.

## Service Topology Checklist

When adding or changing a Symfony service, identify:

- Docker Compose service names for web, cli, database, workers, and protocol-specific runtimes such as RoadRunner or gRPC.
- Service-local `.env` values and root `.env` values that should become shared project configuration.
- Internal base URLs, DSNs, ports, CORS origins, mail/storage endpoints, and inter-service addresses.
- Public REST exposure requirements through `api-gateway/routes.json` and generated nginx/OpenAPI artifacts.
- Database ownership and whether the cli service must be added to `DATABASE_SERVICE_TARGETS`.

## Shared Env Rules

- Prefer the root `.env` for variables shared by multiple services or project-level scripts.
- Keep service-local `.env` files for values that only make sense inside one service.
- Do not duplicate the same source-of-truth value across root and service-local `.env` files.
- Keep internal URLs explicit, for example `CATALOG_SERVICE_BASE_URL=http://catalog-web:8000`, instead of deriving names in scripts.
- After env changes, validate Docker Compose config with `docker compose config --quiet` when practical.

## Database Maintenance Scripts

- The shared database scripts live under `scripts/db-*.sh`.
- The service list is configured through root `.env` as `DATABASE_SERVICE_TARGETS`.
- Format: comma-separated `compose-cli-service:Human label` entries.
- Example: `DATABASE_SERVICE_TARGETS="symfony-cli:Main Symfony service,catalog-cli:Catalog service,cart-cli:Cart service"`.
- When a new Doctrine-backed Symfony service is added, propose adding its cli service to `DATABASE_SERVICE_TARGETS`.
- Do not include fixtures in `db:init`; fixtures are data-destructive and should stay as an explicit command if added later.

## API Gateway Follow-up

If the service exposes public REST endpoints, check whether these need updates:

- `api-gateway/routes.json`
- generated nginx config from `npm run gateway:generate`
- gateway OpenAPI generation
- README generated API endpoint sections via the project README scripts

Generated README sections must not be edited manually.

## Validation

Prefer targeted checks:

- `bash -n scripts/db-common.sh scripts/db-create.sh scripts/db-migrate.sh scripts/db-status.sh scripts/db-init.sh`
- `docker compose config --quiet`
- `node --check` for changed generator scripts
- `npm run gateway:generate` only when gateway route sources changed
- `npm run readme:update` only when generated README content needs refresh
