---
name: avoid-overengineering
description: Keep implementation and refactoring within current requirements. Apply when considering speculative capabilities, compatibility variants, abstractions, or removal of apparently unused defensive code; require approval for optional scope expansion while preserving correctness and useful diagnostics.
---

# Avoid Overengineering

Implement the requested behavior with the smallest maintainable solution supported by current project contracts and usage.

## Additional Functionality

- Before adding a capability outside the task, identify its current caller, documented contract, or concrete requirement. A hypothetical future caller is not evidence of present need.
- If the capability is outside the task and unused by the current project, explain what it enables, why it might be needed, and the additional code or complexity it introduces. Ask for the user's agreement before implementing it.
- Without agreement, omit the optional capability and continue the authorized work that does not depend on it. Silence is not agreement. Do not request approval again for additions already explicitly approved in the session.
- Apply this rule to extra input formats, compatibility paths, configuration options, generic extension points, fallback behavior, and abstractions as well as visible features. Do not introduce them solely because they might be useful later.
- A test written for a new capability does not establish its necessity. Base test scenarios on the requested behavior, existing contracts, or explicitly approved extensions.

## Correctness Is Part of the Task

- Necessary validation, error handling, useful error diagnostics, and boundary-case handling are part of implementing the requested functionality correctly. Do not treat them as optional extensions merely because no direct caller or happy-path example exercises them.
- Before labeling code dead or removing a defensive branch, identify the guarantee it preserves and whether the relevant input or failure is possible under the current contract. Absence of a current test or observed failure does not establish that the branch is unnecessary.
- Prefer simplifying an implementation without weakening its guarantees. Do not shorten useful diagnostics or remove error context solely to reduce line count.
- When necessity is uncertain, inspect the relevant callers and contracts and report the uncertainty. Do not present unverified compatibility assumptions as established requirements, or unverified lack of usage as proof that correctness logic is disposable.
- This skill does not authorize unrelated cleanup or override the project's change-approval process.

## Project Example: API Query Serialization

In `api-client.ts`, accepting absolute URLs in `buildApiUrl()` and accepting both `sectionIds` and `sectionIds[]` as input keys were extra capabilities without a current application requirement. Such additions need an explanation and approval; a test invented to exercise them is insufficient justification.

By contrast, deleting existing query values before appending an array preserves replacement semantics:

```ts
const arrayKey = `${key}[]`;
url.searchParams.delete(key);
url.searchParams.delete(arrayKey);
value.forEach((item) => url.searchParams.append(arrayKey, String(item)));
```

This prevents stale scalar or array values from remaining in the resulting URL, and an empty array clears previous values. Preserve this useful correctness logic rather than classifying it as dead code based only on current callers.

The example is contextual: absolute URLs or alternate formats can be appropriate when a real requirement calls for them. It is not a universal prohibition.
