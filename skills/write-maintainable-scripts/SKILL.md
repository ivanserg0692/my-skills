---
name: write-maintainable-scripts
description: Write or refactor project utility scripts in JavaScript or Bash so their top-level flow is easy to follow. Use for operational scripts that parse arguments, validate configuration, invoke external CLIs, or coordinate several steps; not for ordinary application business code.
metadata:
  short-description: Keep utility script flows readable
---

# Write Maintainable Scripts

Make the script's entry point read as a short sequence of meaningful operations. A reader should see what happens and in what order without first unpacking argument parsing, loops, or CLI details.

- Extract a block when it has a clear responsibility, such as resolving arguments, checking prerequisites, ensuring a resource exists, or synchronizing data. Name the function for the outcome it produces. Keep a small, already clear script inline.
- Keep low-level loops, deduplication, command arguments, and serialization inside the relevant function when they obscure the top-level flow. Use conventional naming for the language: `camelCase` in JavaScript and `snake_case` in Bash.
- Prefer a few focused functions in the current file. Add a shared library only when behavior is genuinely reused or the file has a separate stable responsibility. Avoid wrappers that merely rename one obvious call.
- When a helper returns a restricted subset of resources, make that meaning clear at its call site or with a small focused helper. Do not change a shared API solely to improve one caller's wording.
- Write comments that state the concrete action, check, or reason. Make each comment understandable without requiring the reader to inspect the implementation first. Use direct verbs such as verify, compare, wait, restart, create, or delete, and name what the command checks or changes. Avoid vague phrases such as “get an expected image” when the comment can say “compare the deployed image with the expected image.”

For a readability-only refactor, first identify the existing order of validation, side effects, output, and failure handling. Preserve it, including exit codes, error messages, retries, traps, and secret handling. In Bash, take care with `set -e`, pipelines, and cleanup traps; in JavaScript, preserve exception boundaries and `process.exitCode` behavior. Do not run external commands as part of a structural check when they would mutate infrastructure.

Verify syntax and inspect the diff. Add behavioral checks only when they address a concrete risk introduced by the change.
