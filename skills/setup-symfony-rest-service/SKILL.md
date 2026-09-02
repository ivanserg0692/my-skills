---
name: setup-symfony-rest-service
description: Inspect a Symfony service and recommend missing Composer dependencies and configuration improvements for a production-ready REST API service. Use when planning or checking base REST setup for catalog/product/order microservices, Doctrine ORM, migrations, fixtures, Faker, Serializer, Validator, Nelmio OpenAPI JSON, bundle registration, and configuration review without modifying project code.
---

# Setup Symfony Rest Service

## Workflow

Use this skill to inspect and plan the base dependency setup for a Symfony REST microservice. Do not modify application code, create entities, controllers, migrations, or configuration unless the user separately asks and confirms.

1. Check the workspace `.aiignore` before reading project files.
2. Identify the target service directory and read its `composer.json`.
3. If available, inspect `composer.lock` only to verify already installed transitive packages.
4. Do not recommend packages that are already listed in `require` or `require-dev`.
5. Recommend only packages needed for the requested REST service capabilities.
6. Do not include packages only because they may be installed transitively, or because they are useful but not part of the project baseline REST service setup.
7. Output Composer commands for the user to run manually through the relevant Docker service. Do not run `composer require` yourself.
8. After the user installs packages, inspect `composer.json`, `composer.lock`, and `config/bundles.php` to verify dependency and bundle state.
9. Analyze service configuration files and give configuration recommendations without editing files unless the user separately confirms.
10. If Doctrine recipe added an active generated `DATABASE_URL` in the service `.env` but runtime uses Docker-provided env variables, remove that active service `.env` value as unused noise after confirmation. Keep commented recipe examples unless the user asks to clean them too. Do not copy secrets from shared `.env.local` into the service `.env`.

## Base Dependencies

Runtime packages for a production-ready Symfony REST service:

- `doctrine/orm` - EntityManager, repositories, Unit of Work, and ORM persistence.
- `doctrine/doctrine-bundle` - Symfony integration for Doctrine services and configuration.
- `doctrine/doctrine-migrations-bundle` - versioned database schema migrations.
- `symfony/serializer` - JSON normalization/denormalization for API DTOs and responses.
- `symfony/validator` - validation constraints for request DTOs and domain inputs.
- `nelmio/api-doc-bundle` - OpenAPI JSON generation from PHP attributes and Symfony metadata.

Development-only packages:

- `doctrine/doctrine-fixtures-bundle` - load dev/test catalog data.
- `fakerphp/faker` - generate realistic fixture/test data.

Do not add Swagger UI dependencies when the service only exposes OpenAPI JSON and UI is handled by the API Gateway.

## Command Format

Prefer one command for runtime packages and one command for dev packages. Run commands from the Docker Compose workspace root, not from inside the service directory, when the compose file lives at the workspace root.

Example for `catalog-service` using the `catalog-web` Docker service:

```bash
cd /home/ivan/symfony2026/app

docker compose run --rm catalog-web composer require doctrine/orm doctrine/doctrine-bundle doctrine/doctrine-migrations-bundle symfony/serializer symfony/validator nelmio/api-doc-bundle

docker compose run --rm catalog-web composer require --dev doctrine/doctrine-fixtures-bundle fakerphp/faker
```

If a package is already installed transitively in `composer.lock` but not listed in `composer.json`, explain the difference. Do not recommend adding it explicitly unless the service directly relies on it as part of its own architecture and the project baseline includes it.

## Post-Install Handoff

When outputting Composer install commands, always end with a short reminder that the next step happens after the user runs those commands manually.

Use this exact Russian handoff phrase unless the user asks for another language:

```txt
После выполнения команд вернись с фразой: "установил зависимости, проверь post-install шаги"
```

When the user returns with that phrase or any equivalent message saying dependencies were installed, proceed to the post-install check:

- inspect `composer.json`;
- inspect `composer.lock` for installed/transitive packages;
- inspect `config/bundles.php`;
- inspect the service `.env` for recipe-generated active `DATABASE_URL` noise;
- analyze service configuration and give recommendations;
- report what is correct and what still needs cleanup;
- ask for `делаем` before editing files.

## Related Implementation Skills

For implementing or refactoring REST endpoints that accept JSON request bodies, use `$symfony-rest-request-validation`. Keep this setup skill focused on dependency and configuration readiness.


## Configuration Review

After dependency and bundle verification, inspect the service configuration and give recommendations without editing files unless the user confirms.

Check at least:

- `config/packages/doctrine.yaml` for `DATABASE_URL`, server version, mappings, production cache pools, and test database suffix.
- `config/packages/nelmio_api_doc.yaml` and `config/routes/nelmio_api_doc.yaml` for OpenAPI JSON route, service metadata, and exclusion of documentation endpoints from documented API paths.
- `config/packages/framework.yaml` for REST-service concerns such as session usage and `APP_SECRET`.
- `config/packages/routing.yaml` and the service `.env` for required env variables such as `DEFAULT_URI`.
- `config/packages/validator.yaml` and `config/services.yaml` for baseline REST API readiness.
- Docker Compose and shared `.env.local` only where they affect the service runtime configuration, such as `DATABASE_URL`, service web container, and database healthcheck.

Report findings as recommendations grouped by priority:

- required fixes;
- recommended cleanup;
- production-hardening notes.

## Bundle Verification

Packages that should appear in `config/bundles.php`:

- `Symfony\Bundle\FrameworkBundle\FrameworkBundle`
- `Doctrine\Bundle\DoctrineBundle\DoctrineBundle`
- `Doctrine\Bundle\MigrationsBundle\DoctrineMigrationsBundle`
- `Nelmio\ApiDocBundle\NelmioApiDocBundle`
- `Doctrine\Bundle\FixturesBundle\DoctrineFixturesBundle` for `dev` and `test`

Packages that do not register a bundle and should not be expected in `config/bundles.php`:

- `symfony/serializer`
- `symfony/validator`
- `fakerphp/faker`
- `doctrine/orm`

When reporting verification, separate "installed dependencies" from "registered bundles" so the user can see that component packages are valid even without bundle entries.
