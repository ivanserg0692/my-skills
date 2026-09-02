---
name: feature-documentation-completeness
description: Check feature documentation completeness before closing substantial feature work, especially when a new user-facing, system, inter-service, async, or operational workflow has emerged. Use when Codex is wrapping up a feature, the user says the task is ready/nearly done, asks what remains, asks for an MR description, or the current diff/context suggests README, task docs, generated endpoint blocks, or PlantUML workflow diagrams may be missing. The skill focuses on general feature workflow documentation, not MR-only notes.
---

# Feature Documentation Completeness

## Purpose

Use this skill to decide whether a feature is documented well enough before wrapping up. The goal is not to document every change, but to avoid losing important feature workflows that are hard to infer from code, endpoint tables, or git history.

Treat an MR or git diff as evidence about what changed, not as the target document. The target is project documentation such as `README.md`, `symfony/docs/task-N.md`, dedicated feature docs, generated README endpoint blocks, and PlantUML workflow diagrams.

## Core Workflow

1. Check `.aiignore` before reading project files.
2. Reconstruct what the feature changed from the current context, `git diff --stat`, changed file paths, task docs, README sections, and relevant generated artifacts.
3. Decide whether the feature introduced or changed an important workflow.
4. Check whether that workflow is already described in general project documentation.
5. If documentation is missing or stale, propose minimal documentation changes and wait for the project approval phrase before editing docs.
6. After approval, update only the needed docs/PlantUML sources, regenerate required generated artifacts, and validate formatting.

## Workflow Detection

Consider documentation when the feature introduces or materially changes one of these workflows:

- Client flow: frontend or external client through API Gateway to backend services.
- Authentication flow: login, refresh, logout, `me`, CSRF, JWT validation, trusted identity propagation.
- Business flow: checkout, order creation, status movement, import/export, notification delivery, admin process.
- Inter-service flow: REST, gRPC, Messenger/RabbitMQ, events, snapshots, ownership boundaries.
- Async flow: producer, message, consumer, retry/status transition, currently planned but architecturally relevant components.
- Operational flow: multi-service database init, migrations, fixtures, service bootstrap, or repeatable maintenance process.

Do not force documentation for small local fixes, pure refactors with no behavior change, typo fixes, or implementation details that are already obvious from nearby code.

## Documentation Targets

Prefer the smallest durable documentation target that explains the feature:

- `README.md`: use for important project-level behavior, public usage, architecture notes, store workflows, auth/gateway behavior, and operational workflows.
- `symfony/docs/task-N.md`: use for task-specific scope, status, acceptance notes, implementation contract, and completed subtasks.
- Dedicated docs file: use only when the workflow is too large for README/task docs.
- Generated README endpoint blocks: use for REST endpoints documented from OpenAPI/Swagger sources. Do not write endpoint tables manually.
- MR description: use only when the user asks for MR text; it can summarize docs, but must not replace general documentation.

When a REST API contract changed, prefer existing generated endpoint marker blocks such as:

```md
<!-- START api-endpoints service=... locale=en sourceType=... source=... -->
<!-- END api-endpoints -->
```

This applies to public Gateway endpoints and service-level REST endpoints when they are documented through generated OpenAPI/README blocks. If generated endpoint text is wrong, fix OpenAPI/Swagger source text first and regenerate the README block. Never manually edit generated README sections.

## Diagram Decision

Add or propose a PlantUML diagram only when it explains a real workflow better than prose or a table.

- Use a sequence diagram when order of calls matters: client -> gateway -> service -> gRPC/Messenger -> database/result.
- Use a state diagram when lifecycle states matter: `pending -> paid -> delivered`, failure states, cancellation.
- Use an activity diagram when the business process matters more than HTTP/gRPC mechanics.
- Use a domain relations diagram only when new business entities, ownership boundaries, snapshots, or cross-service relations need explanation.

Do not add component, deployment, context, Docker topology, or broad architecture map diagrams by default. Suggest them only when a new service or infrastructure component makes the topology itself hard to understand.

Store PlantUML files and generated PNGs according to project rules:

- Source: `symfony/docs/plantuml/.../*.puml`
- PNG: `symfony/docs/images/plantuml/.../*.png`
- Markdown: PlantUML marker blocks, not hand-written standalone image links.
- Regenerate with `npm run docs:plantuml` from the repository root.

## Completeness Checklist

Before finalizing substantial feature work, answer these questions briefly:

- What user/system workflow changed?
- Is that workflow understandable from existing README/task docs?
- Are REST endpoints represented through generated README endpoint tags when applicable?
- Does the workflow involve service boundaries, auth, trusted headers, gRPC, Messenger/RabbitMQ, snapshots, or status movement that needs prose?
- Would a sequence, state, or activity diagram reduce ambiguity?
- Are implemented behavior and planned/out-of-scope behavior clearly separated?
- If headings or generated docs changed, did the relevant generator run?

## Approval Gate

Documentation is project documentation. Before editing `README.md`, `docs/`, task docs, PlantUML docs, or MR log files:

1. Show the exact planned documentation changes.
2. Ask for confirmation.
3. Proceed only after the user replies exactly `делаем`.

OpenAPI/Swagger texts are API contract source, not documentation for this approval gate; when code changes affect API contract, update OpenAPI together with implementation as project rules require.

## Related Skills

Use this skill as a documentation completeness layer, not as a replacement for narrower project skills:

- Use `documentation-toc` when adding or changing Markdown headings/TOC blocks.
- Use `service-maintenance` for shared env, internal URLs, Docker service onboarding, database maintenance targets, gateway generation inputs, and service maintenance scripts.
- Use `create-merge-request-description` when the user asks for MR/PR description text.
- Use `create-project-task` when creating or formalizing task docs.

Do not duplicate those skills inside this one. This skill should decide whether feature workflow documentation is missing and route the work to the right documentation artifact.
