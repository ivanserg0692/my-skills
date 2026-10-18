---
name: environment-isolation
description: Preserve the independence of dev, prod, and test-load and isolate configuration consumers when designing architecture or changing infrastructure, environment variables, Docker Compose, or Kubernetes deployments.
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

## Configuration and Environment Variables

- Pass each variable only to the components that use it. Variables used exclusively by Kubernetes deployments in `prod` or `test-load` must not automatically enter Docker Compose containers, Symfony, or other components. Variables used only by Docker Compose must not enter Kubernetes without a concrete need.
- Share variables between Docker Compose and Kubernetes only when both actually use them. Avoid unnecessary configuration duplication and never pass secrets to components that do not use them.
- When adding or changing a variable, trace its full propagation through `.env.local`, `env_file`, configuration generators, PHP process environments, Kubernetes Secrets, and ConfigMaps. Check every consumer and forwarding mechanism; a variable's intended purpose does not limit where it is actually injected.
- Do not add Kubernetes-only variables to a shared `.env.local` that is passed wholesale to Docker Compose containers. Use an isolated configuration source or an explicit filtering or selection mechanism that prevents delivery to unrelated components. Keep real secrets in the repository-root `.env.local` as required by project policy; when it supplies Kubernetes-only secrets, exclude them from unrelated container environments.

Example of a violation: `FILEBEAT_PASSWORD` is needed only by Kubernetes, but a shared `env_file` passes it to 15 Docker Compose containers. Apply the same isolation rule to every variable and infrastructure component.
