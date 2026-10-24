---
name: go-polymorphism
description: Design, review, or refactor Go behavior contracts using interfaces, composition, and constructor injection. Apply when deciding dependency boundaries, consumer-owned interfaces, DIP and ISP, or whether dispatch layers and forwarding methods improve the design. Keep concrete types, functions, and switches when they are clearer.
---

# Go Polymorphism

Choose the smallest behavior contract that protects a useful boundary. Keep mutable state with its owner and shared coordination with the application component that owns the relevant invariants. Apply this skill to Go projects independently of their framework or domain.

## Distinguish the concepts

| Concept | Meaning in Go | What it does not establish by itself |
| --- | --- | --- |
| Polymorphism | A consumer invokes an interface method and the dynamic concrete value supplies the implementation. | Dependency construction, correct ownership, or an inverted dependency direction. |
| Dependency Injection (DI) | A component receives dependencies from outside, usually through its constructor. Dependencies can be concrete values, interfaces, or functions. | DIP: injecting a concrete infrastructure client is still a concrete dependency. |
| Dependency Inversion Principle (DIP) | Application policy defines or consumes a behavior contract; infrastructure implements it without the policy depending on infrastructure details. | Merely replacing a concrete field with an interface exposing the same infrastructure API. |
| Interface Segregation Principle (ISP) | Each consumer depends on the operations it actually needs. | One interface per method regardless of meaning, or a single interface covering the entire application. |

These mechanisms can work together: a consumer defines a focused interface, an application composition point injects an implementation, and method calls use polymorphism. Assess each property separately rather than treating an interface as proof of all four.

## Understand Go's interface model

Go interfaces are satisfied implicitly through method sets; implementations do not declare `implements`. For ordinary behavior interfaces, structural matching means the required method names and signatures must match. Parameter names do not affect compatibility. Matching method sets establish type compatibility, not equivalent semantics: document errors, effects, and ownership when callers rely on them.

- A type `T` has methods declared on `T`; `*T` has methods declared on both `T` and `*T`. Pointer receiver methods therefore usually require a pointer when assigning to an interface. Automatic address-taking for an addressable method call does not make `T` satisfy that interface.
- Choose receivers according to mutation, identity, and copying requirements. Do not change a receiver merely to satisfy an unnecessary abstraction.
- Use an assertion such as `var _ Saver = (*DiskStore)(nil)` where an intentional contract deserves an explicit compile-time check. Assertions are optional and must not introduce a reverse import solely to perform the check.
- Embed interfaces to combine meaningful capabilities. Interface embedding combines requirements; it does not inherit an implementation. Struct embedding promotes methods and can unintentionally expose capabilities or satisfy additional interfaces; prefer named fields when that exposure is unwanted.
- A nil interface differs from an interface containing a typed nil pointer. Return a plain `nil` for absent interface results, especially errors. Define whether a dependency is required or optional; avoid reflection or generic nil-detection helpers as a default solution.
- Type-parameter constraint interfaces and generic algorithms solve a different problem from runtime behavior injection. Introduce generics only when a real type relationship or reusable algorithm calls for them.

## Establish ownership and dependency direction

Before changing dispatch, identify local state, cross-component state, service effects, and the component responsible for each invariant. Moving branches into helpers that receive the whole application model or service container does not improve encapsulation.

Prefer defining an interface in the package that consumes its behavior. Implementations can satisfy it without importing that package. Return concrete implementation types from constructors when useful, and accept the narrow interface at the consuming boundary. Shared, established contracts such as standard library interfaces can remain in their existing package.

Keep interfaces next to the feature or consumer they serve. Avoid creating a broad `interfaces` package or a new package solely to break an import cycle. First reconsider dependency direction and whether infrastructure types have leaked into the contract. Introduce shared packages only for concepts with an actual shared responsibility.

Coordinate persistence, navigation, cancellation, transactions, and resource ownership in the component that owns those operations. Consumers can request actions through focused methods without receiving the coordinator's fields or concrete type.

## Inject focused dependencies

Use constructors to make required dependencies visible. Store them in unexported fields typed to the operations the consumer uses. Assemble implementations at an application entry point or other existing composition point; no DI framework or registry is required.

Group methods according to actual consumer needs and coherent responsibilities. A component may accept one small multi-method interface or several independent dependencies. Embed capabilities when that improves reuse or expresses a real combined requirement; a hierarchy of single-method interfaces is not a goal.

An implementation can have many methods while each consumer sees only a small subset. Adding a method for one consumer should not enlarge unrelated consumers' contracts. Export a method when another package intentionally needs it, not merely because interfaces are being introduced.

Constructor injection does not require executing the dependency during construction. When two objects refer to each other, construct the coordinator, attach dependent components, and complete initialization before invoking callbacks or publishing the objects. Object reference cycles are different from package import cycles; avoid partial-initialization callbacks and document shutdown ownership when relevant.

## Choose the mechanism

| Choice | Prefer when | Reconsider when |
| --- | --- | --- |
| Concrete type | Its implementation is part of the useful API and no behavior boundary needs abstraction. | Consumers acquire infrastructure knowledge or unrelated capabilities. |
| Interface | A consumer needs a focused behavior boundary, substitutable implementations, or isolation from implementation details. | It merely mirrors a struct's methods without protecting a boundary. |
| Function | A dependency is naturally one operation or a small callback with no useful object protocol. | Many unrelated callbacks obscure a coherent dependency or lifecycle. |
| Composition | A component combines existing capabilities while preserving their ownership. | Embedding widens its public surface or hides which component performs effects. |
| Switch or type switch | A small stable decision, external protocol adaptation, validation, or state transition reads clearly. | The same variant discriminator spreads through unrelated behavior or variants own independently evolving logic. |

A useful interface can have one current implementation; do not invent a second production implementation to justify it. Conversely, mockability alone does not require an interface for every struct. Prefer fakes that exercise the actual boundary over tests coupled to internal forwarding calls.

## Remove unnecessary dispatch and forwarding

Compare the full call path before adding handlers. A path such as consumer → request object → dispatcher → receiver → coordinator may be simplified to consumer → injected interface when the request is synchronous and carries no independent lifecycle or meaning.

Read [the worked examples](references/examples.md) when comparing constructor injection, composed interfaces, typed nil behavior, or removal of a dispatch layer.

Do not turn this simplification into a ban on messages. Keep data messages or explicit dispatch when they serve a queue, asynchronous completion, serialization, recording, retries, or a meaningful protocol boundary. A switch for framework-owned messages can coexist with interface-based application dependencies.

For each short method, ask what it contributes:

- Merge a method that only forwards identical arguments and results, when one implementation can also satisfy the public contract. Rename the implementation and update internal callers instead of keeping a public wrapper solely to reach a private method.
- Inline a private single-use forwarding helper when it adds no useful meaning or boundary.
- Keep methods that change state, validate, translate errors, manage resources, or intentionally adapt an external contract. Length is not the criterion.
- Do not redirect all callers through a method that adds behavior they did not previously invoke. A refresh action that clears an error differs from a loader also used during initialization.
- When working on an existing public API, account for external callers before renaming or removing methods. A compatibility adapter may have a real purpose.

## Preserve behavior during refactoring

Replacing returned requests with direct interface calls changes where execution happens. Identify which actions run synchronously and which effects are deferred before rewriting the path.

Preserve the order of local state updates, coordinator calls, navigation, validation, and returned work. Return immediately after navigation when the original path stopped processing the event. Do not execute deferred commands inside event handlers merely because injection makes the dependency available.

Retain busy/input gates, cancellation propagation, stale-result checks, error updates, resource cleanup, and batching or scheduling semantics when they exist. Keep asynchronous completions as data if the application relies on processing them in its event loop. Do not move state mutation into background callbacks without an explicit concurrency design.

Verify observable behavior using the project's existing checks. Where needed, test synchronous state changes separately from deferred effects, cancellation, errors, and ordering. Update tests tied to a removed request representation to assert equivalent behavior rather than preserving obsolete plumbing.

## Detect excess architecture

Reconsider a proposal if it introduces a handler struct per trivial action, dozens of function dependencies, generic adapters, registries, controllers, or packages primarily to eliminate a switch. Also look for consumer interfaces exposing a service container, repeated assertions back to concrete types, and forwarding layers whose only purpose is to satisfy one another.

Before implementing, explain the proposed ownership boundary, why the selected contract is useful, which dispatch layers disappear, and what complexity remains. Prefer fewer concepts and a clear call path over a nominally switch-free design. Respect the project's approval rules and preserve behavior unless a change is explicitly authorized.
