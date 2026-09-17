---
name: information-expert
description: Use when generating, reviewing, or refactoring application or domain code where a service or helper extracts another object's data to calculate, merge, validate, or transform that object's state. Apply GRASP Information Expert to decide whether the behavior belongs on the data-owning object; keep use-case orchestration in services.
---

# Information Expert

Assign an operation to the object that has the information needed to perform it, when that responsibility fits the object's meaning. Use this principle to improve cohesion and reduce how much other classes know about the object's structure. A DTO label alone is not a reason to keep meaningful behavior outside the object.

## Look for misplaced responsibility

Check whether a service or helper:

- reads several fields or nested items from another object only to calculate or update a result;
- builds temporary arrays to group, merge, or recalculate that object's data;
- depends on field layout, ordering, uniqueness, or other rules owned by that object;
- repeats knowledge of the same structure across multiple callers;
- exists mainly to keep a result object passive.

These are signals to investigate, not proof that the code must change. A simple local projection may still be clearer as a loop.

For example, if `merge($resultA, $resultB)` needs to unpack both results, consider `$resultA->merge($resultB)` when a result can enforce the merge rules using its own state and the other result's public contract. Do not move behavior merely to replace a short loop with an extra method.

## Choose the owner

Ask: **Which object has the information required for this operation, and is its responsibility currently outside that object?**

Prefer behavior on the data-owning object when the operation:

- naturally describes that object's state or meaning;
- uses information the object already owns or can obtain through the other object's public contract;
- can protect its invariants in one place;
- reduces callers' dependence on its internal representation;
- does not need to coordinate a broader use case.

A result that gains natural behavior may become a Result Object rather than a passive DTO. This is a possible consequence of applying Information Expert, not a requirement to replace every DTO. Use an existing object when it is the right owner; do not introduce a service, helper, merger, or collection merely to preserve passivity.

Keep orchestration in an application or domain service when the operation needs repositories, persistence or transactions, HTTP/gRPC/API calls, a message broker, infrastructure dependencies, or coordination among several independent models. Do not make a result object responsible for those collaborators just to place more logic inside it.

## Review and refactor

When reporting a candidate, identify the class that currently knows too much, the information it extracts, and the proposed Information Expert. Explain how the change affects cohesion and coupling, and describe trade-offs such as mutability, API surface, allocation, and behavior preservation where relevant. If ownership is ambiguous or the existing code is clearer, say so rather than forcing a move.

Before editing, respect the project's approval and refactoring boundaries. Preserve observable behavior, ordering, side effects, persistence timing, and external contracts unless the user has approved a change. Add a public method only when another class needs it as an intentional API; keep local implementation details private.
