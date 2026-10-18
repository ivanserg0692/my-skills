---
name: filebeat-testing
description: Test Filebeat configuration changes in an isolated Filebeat 9.5.2 Docker sandbox before proposing them for Kubernetes. Use for changes to Filebeat inputs, processors, filters, parsing, field mappings, event routing, or other event processing settings.
metadata:
  short-description: Test Filebeat changes in isolation
---

# Filebeat Testing

Use this skill whenever a task changes Filebeat event collection or processing. The bundled sandbox is the reusable test harness; keep it independent of Kubernetes and Elasticsearch.

## Location and harness

- The harness is in `sandbox/` next to this `SKILL.md`.
- It runs the official Filebeat `9.5.2` image using Docker Compose, console output, and no Elasticsearch or Kubernetes.
- `sandbox/filebeat.yml` is the working dissect example. `sandbox/filebeat-conflict.yml` preserves the original `host` collision scenario.
- Each Compose service has its own `path.data` and temporary filesystem for registry state. Use `docker compose run --rm` from `sandbox/` for fresh, repeatable runs.
- Sample data is under `sandbox/samples/`. Keep representative test events there, and add cases for each changed behavior.

## Required workflow

1. Read the current Filebeat configuration and identify the exact inputs, processors, filters, outputs, fields, and event paths affected. Inspect real log examples when available. Keep secrets out of test fixtures and output.
2. Create representative test events from the actual input format, including valid, edge, and malformed cases relevant to the change. Preserve a failing regression case when fixing a known issue.
3. Exercise the candidate configuration in this sandbox before proposing changes to the primary Filebeat configuration. Adapt the harness or add a dedicated scenario when the default sample is not representative; keep its Compose paths and state isolated.
4. Run Filebeat configuration validation and execute all relevant success and failure scenarios with console output. For processors under test, make failures observable (for example, use `ignore_failure: false` in a test configuration) and avoid filters such as `drop_event` that hide intermediate results until those filters themselves are being tested.
5. Confirm events are not lost because of input selection, `include_lines`/`exclude_lines`, processor errors, `drop_event`, or field conflicts. Verify expected parsed fields, their types, and that sensitive values such as tokens, credentials, and query parameters are absent or masked as required.
6. Report commands and observed output, including remaining gaps. Only after successful relevant tests, propose the minimal primary-configuration change. Do not apply or deploy Kubernetes changes without separate explicit user approval.

## Run the bundled regression cases

From this skill's `sandbox/` directory:

```sh
docker compose run --rm conflict
docker compose run --rm fixed
```

The first case should reproduce `cannot override existing key with \`host\``. The second should parse the sample and expose `nginx_time` and `nginx_host` while retaining ECS `host.name`. Stop each run after the event or processor error appears. Review the console output and report the relevant fields/error rather than assuming a successful container start means successful event processing.

For a changed configuration, add or adapt a Compose scenario and a matching config file rather than overwriting the regression cases. Give every service a unique `path.data` and temporary registry location.

## Replacing samples and extending tests

- Put new log lines in a file under `sandbox/samples/` matching the input glob. The bundled filestream fingerprint requires at least 64 bytes in the file.
- Keep the original sample when it remains a useful regression case; use a separate file or input glob for new formats.
- Adjust input paths, processors, and console output in a scenario-specific Filebeat config. Do not add Elasticsearch or Kubernetes dependencies to this harness.
- Use separate Compose service names and `path.data` values when testing distinct configurations. `docker compose run --rm` plus the configured `tmpfs` gives each run fresh registry state.
- Clean up any persistent resources created during experiments with `docker compose down --volumes --remove-orphans` from `sandbox/`.

## Reporting

State which scenario and sample were run, whether config validation passed, what fields appeared, whether events were dropped or processors failed, and whether sensitive values were absent. If testing fails, give the exact stage and observable error. Keep sandbox findings separate from changes to the primary configuration. Kubernetes deployment is a separate approval step.
