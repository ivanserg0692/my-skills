---
name: project-change-approval
description: Apply the project's mandatory plan-and-confirmation gate before creating, editing, deleting, or generating any project files, including code, configuration, skills, and documentation.
---

# Project Change Approval

Apply this skill before any operation that changes project files. Read-only inspection and analysis may proceed before approval.

1. Show the user a concrete plan before changing files. Name the files or areas to change, describe the intended edits and their purpose, and identify any behavior-sensitive changes or generated files. State any material uncertainty that affects the plan.
2. Ask for confirmation and wait for the user's reply. Proceed only when the user replies exactly `делаем`. A general expression of agreement or the original task request does not replace this confirmation.
3. Apply only the confirmed plan. If the scope materially changes, explain the revised plan and wait for another `делаем` before the additional changes.

The gate covers direct edits and commands that write project files, including generators, formatters, documentation renderers, and scripts that update tracked or untracked files. Do not run such commands before confirmation.

## Documentation

Before editing project documentation, show the planned documentation changes explicitly and wait for `делаем`. Treat `docs/`, `README.md`, and merge request descriptions as project documentation. Do not treat OpenAPI/Swagger operation texts as documentation for this separate confirmation step. When an approved code change affects the API contract, update the OpenAPI/Swagger texts with the implementation without requesting a second documentation confirmation. Edit other documentation only after its planned changes have been confirmed.
