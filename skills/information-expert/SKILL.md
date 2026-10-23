---
name: information-expert
description: Use when designing, reviewing, or refactoring state ownership in application, domain, or UI components in any language, including Go and PHP. Identify behavior that belongs with its data, helpers that unpack another object's state, and models that combine independently evolving responsibilities; keep shared operations in the coordinating component.
---

# Information Expert

Assign an operation to the object that has the information needed to perform it, when that responsibility fits the object's meaning. Use this principle to improve cohesion and reduce how much other classes know about the object's structure. A DTO label alone is not a reason to keep meaningful behavior outside the object.

Apply this guidance across languages and frameworks. PHP examples illustrate ownership decisions, not a requirement to use classes or inheritance. In Go, use structs, methods, composition, and small consumer-owned interfaces where they improve a real contract.

## Component state ownership

A single model may already own all the data while combining several independently evolving responsibilities. Check whether local state, event handling, validation, and rendering naturally belong to separate components, especially in UI roots and state machines. Apply SRP to those responsibility boundaries; keep shared operations and lifecycle coordination with the root or application service.

Apply this check during initial design, before a large model or helper exists, as well as during review and refactoring. When behavior also varies by type, mode, or component identity, apply [polymorphism](../polymorphism/SKILL.md) to compare conditional dispatch with behavioral contracts. This skill determines ownership; the polymorphism skill guides dispatch choices. Use both when both concerns are present.

Extraction should move relevant state and behavior together. A handler that receives the entire root model or service container preserves the original coupling. Prefer focused value snapshots, events, and requests; retain the ordering of synchronous updates and asynchronous effects. Do not introduce interfaces for every struct, a component registry, or a package per screen without a concrete need.

For example, a Go form component can own its text inputs, focus, and draft configuration, implement `update(tea.Msg)` and `view()`, and return a typed save request. The root handles persistence and cancellation and returns operation results through an explicit contract. Passing `*rootModel` into `form.update` would leave responsibility for the form's internals spread across components. In Go, unexported fields provide package-level access control; grouping components in one package also relies on respecting their method contracts.

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
