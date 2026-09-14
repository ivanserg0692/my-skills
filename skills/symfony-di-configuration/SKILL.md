---
name: symfony-di-configuration
description: Decide which Symfony dependency-injection wiring belongs in services.yaml and which can be expressed with PHP attributes, while keeping environment configuration centralized and preserving container behavior.
metadata:
  short-description: Keep Symfony DI configuration consistent
---

# Symfony DI Configuration

Use this skill when reviewing or refactoring Symfony service wiring between
`config/services.yaml` and PHP dependency-injection attributes.

## Project convention

- Keep values that come from environment variables in `config/services.yaml`.
  This includes `env(...)` bindings for credentials, URLs, feature flags,
  numeric limits, index names, lock IDs, and other deploy-time settings.
- Treat `services.yaml` as the central source of truth for environment-backed
  configuration. Do not scatter `#[Autowire(env: ...)]` declarations through
  application classes unless the user explicitly requests an exception.
- Prefer PHP attributes for structural container metadata that does not own a
  deploy-time value, for example `#[AsAlias]`, `#[AsDecorator]`,
  `#[AutowireDecorated]`, and explicit service references such as
  `#[Autowire(service: 'app.some_named_service')]`.
- Keep infrastructure-only service definitions, factories, aliases that need
  dynamic selection, and test-specific service replacements in YAML when the
  container configuration is clearer or requires a service expression.

## Decision procedure

1. Inspect the existing `services.yaml`, the target class constructors, and
   nearby project conventions before moving wiring.
2. Classify each argument or definition:
   - environment-backed value → keep the binding in YAML;
   - structural relationship between services → consider a PHP attribute;
   - dynamic factory, conditional alias, or test override → usually keep in
     YAML;
   - named service reference without an environment value → attribute is
     acceptable when it improves locality and does not duplicate YAML.
3. Preserve the current service ID, alias, decoration order, laziness, and
   runtime value types. A configuration refactor must not change behavior.
4. Remove both sides of a moved binding: do not leave a YAML binding and a
   constructor attribute competing for the same argument.
5. Keep related environment bindings grouped in the central YAML defaults
   section rather than reintroducing one-off `%env(...)%` expressions.

## Common mappings

| Requirement | Preferred location |
| --- | --- |
| `CATALOG_READ_MODEL`, feature flag, URL, password, index alias | `services.yaml` env binding |
| Interface implemented by one concrete service | `#[AsAlias]` or YAML alias |
| Entity manager / service decoration | `#[AsDecorator]` and `#[AutowireDecorated]` |
| A decorator's wrapped service | `#[AutowireDecorated]` |
| Selecting a named, already-defined service | `#[Autowire(service: ...)]` when no env value is involved |
| Factory result selected by an env value | Factory service and env binding in YAML |
| Dynamic service closure / conditional backend | YAML factory wiring or `#[AutowireServiceClosure]` only when it does not hide configuration |

## Validation

After changing wiring, run targeted checks in the project Docker environment:

- `docker compose exec -T catalog-cli php bin/console lint:container`
- the affected unit/functional test suites;
- `docker compose config --quiet` when Compose or env wiring changed;
- `git diff --check`.

Also inspect the compiled container errors for duplicate bindings, unresolved
service IDs, invalid scalar casts, and changes to aliases or decorators.
