---
name: polymorphism
description: Use when designing, implementing, reviewing, or extending behavior that varies by type, mode, strategy, or component identity in any language. Compare conditional dispatch with polymorphism and composition, especially when the same discriminator recurs or variants evolve independently.
---

# Polymorphism

Apply GRASP Polymorphism to assign variant-specific behavior to components behind a meaningful common contract. Use Replace Conditional with Polymorphism when it improves cohesion and makes changes local. Apply this guidance during initial design as well as refactoring; a large existing switch is not a prerequisite.

## Assess the variation

Identify what selects the behavior, which operations vary, and which state, dependencies, and invariants belong to each variant. Look for:

- the same discriminator repeated across event handling, validation, rendering, or execution;
- variants that evolve independently or own different state and collaborators;
- a new variant requiring edits to several unrelated handlers;
- a useful shared operation whose implementation naturally belongs to each variant.

These are signals to compare designs, not instructions to extract an interface. For new code, use concrete requirements and expected extensions rather than speculative future variants. Apply [information-expert](../information-expert/SKILL.md) when ownership also needs attention: moving branches into handlers that receive the entire root model or service container leaves the coupling in place.

## Choose the smallest useful contract

Compare explicit conditional dispatch with components that own their state and implement the varying operation. Use OCP to assess whether adding a variant changes only its implementation and necessary construction or registration, rather than spreading edits across unrelated behavior. Keep shared operations and cross-component coordination with the application service or root model.

Adapt the mechanism to the language: Go structs and small consumer-owned interfaces, PHP interfaces and composition, or functions and closures when behavior needs no independent state. Do not require inheritance, an interface for every type, a registry, or a package per variant. A function or lookup table may be sufficient for a simple mapping.

Keep conditionals when they clearly express:

- a small, stable choice with little independent variant behavior;
- validation, ranges, or data mapping;
- protocol parsing or selection of a concrete component at a boundary;
- state transitions, navigation, or coordination whose rules belong together.

A sealed set of variants with exhaustive matching may be clearer when new operations are common and new variants are rare. Consider that trade-off alongside the cost of adding variants. Polymorphic components can coexist with an explicit state machine; eliminating every switch is not the objective.

Before substantial implementation, briefly explain the actual variation, the proposed contract, where state belongs, and why the chosen approach is clearer. A decision to retain a conditional is a valid outcome.

## Preserve behavior during extraction

Move relevant state and behavior together. Pass focused values, events, or requests across component boundaries instead of exposing the coordinator's internals. Preserve public contracts, synchronous update order, asynchronous effects, errors, cancellation, and other observable behavior. Respect project approval rules; identifying an architectural opportunity does not authorize unrelated refactoring.

Verify meaningful existing behavior and variant-specific paths with checks appropriate to the change. Report what moved and why, and identify deliberate conditionals that remain.

For example, independently evolving UI screens may own their inputs, focus, event handling, and rendering, return typed requests, and receive operation results from the root. The root can still dispatch shared navigation and persistence requests explicitly. A short status-to-label switch can remain a switch.

## Keep serialization rules at the boundary

Selecting an input data shape is distinct from executing application behavior. For Symfony request/response DTOs, apply [dto-and-validation](../dto-and-validation/SKILL.md): retain Serializer discriminator mapping and validation rules. Do not put payment execution, persistence, or other service operations on a request DTO merely because it has variants. Symfony attributes are not a requirement for other languages or frameworks.
