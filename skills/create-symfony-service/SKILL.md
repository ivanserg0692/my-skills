---
name: create-symfony-service
description: Provide the exact Docker-based Symfony CLI command for creating a new Symfony microservice in this workspace. Use when the user wants to add, scaffold, create, or plan another Symfony service or microservice, especially catalog, product, order, auth-adjacent, or other backend services. The agent must only print commands for the user to run manually and must not execute symfony new, composer install/update/require, Docker builds, or container startup.
---

# Create Symfony Service

## Overview

Use this skill to answer requests about creating a new Symfony microservice in `/home/ivan/symfony2026/app` with the existing Docker `symfony-cli` service.

The agent must provide commands only. Do not run the service creation command yourself.

## Required Behavior

1. Check `.aiignore` before inspecting project files.
2. If the user asks to create a new service, give the exact command for the user to run manually.
3. Do not run `symfony new`, `composer install`, `composer update`, `composer require`, `docker compose up`, `docker compose build`, or `docker compose pull`.
4. Prefer a clean Symfony Skeleton by default: `symfony new <service-name> --no-git`.
5. Use `--webapp` only if the user explicitly asks for a full web application scaffold.
6. Explain that `--no-git` avoids creating a nested Git repository, which would otherwise appear in the parent repository as a gitlink/submodule-like entry.

## Command Template

Use this command from the repository workspace root:

```bash
cd /home/ivan/symfony2026/app
docker compose run --rm --no-deps \
  -v "$PWD:/workspace-root" \
  -w /workspace-root \
  symfony-cli \
  symfony new <service-name> --no-git
```

Replace `<service-name>` with the requested service directory name, for example `catalog-service` or `product-service`.

## Example Output

For a catalog service, provide:

```bash
cd /home/ivan/symfony2026/app
docker compose run --rm --no-deps \
  -v "$PWD:/workspace-root" \
  -w /workspace-root \
  symfony-cli \
  symfony new catalog-service --no-git
```

For a product service, provide:

```bash
cd /home/ivan/symfony2026/app
docker compose run --rm --no-deps \
  -v "$PWD:/workspace-root" \
  -w /workspace-root \
  symfony-cli \
  symfony new product-service --no-git
```

## After-Creation Git Check

Tell the user to verify how Git sees the new directory:

```bash
cd /home/ivan/symfony2026/app
git status --short --untracked-files=all
```

If Git shows the service as a single `A  <service-name>` entry or reports paths as being in a submodule, tell the user to fix the index:

```bash
cd /home/ivan/symfony2026/app
git rm --cached <service-name>
git add <service-name>
```

This removes the gitlink from the parent index and adds the service files normally. It does not delete the service files from disk.

## Infrastructure Extension Template

When the user asks how to extend infrastructure for a newly created service, provide the template but do not run Docker startup, build, pull, or dependency installation commands.

Use the existing workspace pattern:

- Add service-specific infrastructure to `/home/ivan/symfony2026/app/docker-compose.yml`.
- Put service credentials and DSNs in the shared `/home/ivan/symfony2026/app/.env.local` for simplicity.
- Use `env_file` for the Compose service:

```yaml
env_file:
  - .env.local
```

For a PostgreSQL database container, still include explicit `environment` mapping because the official PostgreSQL image initializes itself from `POSTGRES_DB`, `POSTGRES_USER`, and `POSTGRES_PASSWORD`:

```yaml
<service-name>-db:
  image: postgres:16-alpine
  container_name: <service-name>-db
  env_file:
    - .env.local
  environment:
    POSTGRES_DB: ${<PREFIX>_POSTGRES_DB:-<db-name>}
    POSTGRES_USER: ${<PREFIX>_POSTGRES_USER:-<db-user>}
    POSTGRES_PASSWORD: ${<PREFIX>_POSTGRES_PASSWORD:-<db-password>}
  ports:
    - "<host-port>:5432"
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -d <db-name> -U <db-user>"]
    timeout: 5s
    retries: 5
    start_period: 30s
  volumes:
    - <service-name>_database_data:/var/lib/postgresql/data
```

Also add the volume:

```yaml
volumes:
  <service-name>_database_data:
```

Add matching variables to `/home/ivan/symfony2026/app/.env.local`:

```env
#<service-name> configs
<PREFIX>_POSTGRES_DB=<db-name>
<PREFIX>_POSTGRES_USER=<db-user>
<PREFIX>_POSTGRES_PASSWORD=<db-password>
<PREFIX>_DATABASE_URL=postgresql://<db-user>:<db-password>@<service-name>-db:5432/<db-name>?serverVersion=16&charset=utf8
```

Important Symfony nuance: a variable such as `<PREFIX>_DATABASE_URL` is not used automatically by Doctrine if the service expects `DATABASE_URL`. Later, either pass it as `DATABASE_URL` into the service runtime container or configure that service's Doctrine connection to use `%env(<PREFIX>_DATABASE_URL)%`.

For local HTTP/REST access, add a web service that inherits from `symfony-base`, mounts the new service directory, and exports `<PREFIX>_DATABASE_URL` from `env_file` as the standard `DATABASE_URL` expected by Symfony and Doctrine at container runtime:

```yaml
<service-name>-web:
  <<: *symfony-base
  container_name: <service-name>-web
  volumes:
    - ./<service-name>:/workspace
  depends_on:
    <service-name>-db:
      condition: service_healthy
  command:
    - sh
    - -lc
    - export DATABASE_URL="$$<PREFIX>_DATABASE_URL"; /root/.symfony5/bin/symfony server:stop || true; /root/.symfony5/bin/symfony serve --allow-http --no-tls --listen-ip=0.0.0.0 --port=8000 --daemon; exec tail -f /dev/null
  ports:
    - "<host-web-port>:8000"
  stdin_open: false
  tty: false
```

Use `$$<PREFIX>_DATABASE_URL` in the Compose command so Docker Compose does not interpolate the value on the host. The variable comes from `env_file` inside the container.

For a catalog service, use:

```yaml
catalog-db:
  image: postgres:16-alpine
  container_name: catalog-db
  env_file:
    - .env.local
  environment:
    POSTGRES_DB: ${CATALOG_POSTGRES_DB:-catalog}
    POSTGRES_USER: ${CATALOG_POSTGRES_USER:-catalog}
    POSTGRES_PASSWORD: ${CATALOG_POSTGRES_PASSWORD:-catalog}
  ports:
    - "5433:5432"
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -d catalog -U catalog"]
    timeout: 5s
    retries: 5
    start_period: 30s
  volumes:
    - catalog_database_data:/var/lib/postgresql/data
```

```env
#catalog configs
CATALOG_POSTGRES_DB=catalog
CATALOG_POSTGRES_USER=catalog
CATALOG_POSTGRES_PASSWORD=catalog
CATALOG_DATABASE_URL=postgresql://catalog:catalog@catalog-db:5432/catalog?serverVersion=16&charset=utf8
```

```yaml
catalog-web:
  <<: *symfony-base
  container_name: catalog-web
  volumes:
    - ./catalog-service:/workspace
  depends_on:
    catalog-db:
      condition: service_healthy
  command:
    - sh
    - -lc
    - export DATABASE_URL="$$CATALOG_DATABASE_URL"; /root/.symfony5/bin/symfony server:stop || true; /root/.symfony5/bin/symfony serve --allow-http --no-tls --listen-ip=0.0.0.0 --port=8000 --daemon; exec tail -f /dev/null
  ports:
    - "8010:8000"
  stdin_open: false
  tty: false
```

## Notes

- The Docker Compose service is `symfony-cli`.
- The default Compose mount maps `./symfony` to `/workspace`, so the command adds an extra `-v "$PWD:/workspace-root"` mount to create sibling services under `app/`.
- Use `--no-deps` because creating the project does not require starting database, RabbitMQ, Mailpit, MinIO, or other Compose dependencies.
- Keep the answer short and command-focused unless the user asks for architecture or Docker integration steps.
