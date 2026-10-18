---
name: environment-isolation
description: Preserve the independence of dev, prod, and test-load when designing project architecture, service dependencies, infrastructure, or Kubernetes deployments.
---

# Environment Isolation

The project has three independent environments:

| Environment | Purpose | Kubernetes cluster |
| --- | --- | --- |
| `dev` | Development | Local kind cluster `symfony2026-dev` |
| `prod` | Production operation | Separate existing Kubernetes cluster; namespace `symfony2026` |
| `test-load` | Load testing | Separate k3s cluster; namespace `symfony2026-load-test` |

The `test-load` environment uses the `load-test` profile in project configuration.

Environments are deployed and used separately. Do not assume they run simultaneously. Each environment must contain the services and data it needs. Do not design dependencies on another environment's services, storage, or internal addresses. Starting and operating one environment must not require another environment to be available.
