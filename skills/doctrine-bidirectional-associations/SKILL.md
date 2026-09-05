---
name: doctrine-bidirectional-associations
description: Review or implement Doctrine ORM bidirectional association methods in Symfony entities. Use when changing generated entity setters, add/remove collection methods, mappedBy/inversedBy relations, or fixing an in-memory owning/inverse side inconsistency. Do not use for repository fetch joins or query optimization.
---

# Doctrine Bidirectional Associations

## Core Rule

Doctrine persists a bidirectional association according to its owning side. It does not automatically keep both sides of the already loaded PHP object graph synchronized.

Symfony Maker may generate synchronization in collection `add...()` and `remove...()` methods while leaving a `ManyToOne` setter as a direct property assignment. Inspect the actual methods instead of assuming generated entities are symmetric.

## Workflow

1. Read both related entities and identify the owning side from `inversedBy`; identify the inverse side from `mappedBy`.
2. List every public method through which callers can change the relation.
3. Choose the existing domain method as the canonical mutation path. Do not add public methods only to make synchronization convenient.
4. Ensure every existing public mutation path updates the owning side and leaves both objects consistent in memory.
5. Preserve mapping options, cascade behavior, orphan removal, nullability, persistence timing, and domain behavior unless their change was explicitly approved.
6. Test assignment, reassignment, repeated calls, removal, and each supported public entry point.

## To-One Setter Pattern

For a nullable owning-side `ManyToOne` setter:

```php
public function setParent(?ParentEntity $parent): static
{
    if ($this->parent === $parent) {
        return $this;
    }

    $previousParent = $this->parent;
    $this->parent = $parent;

    $previousParent?->removeChild($this);

    if ($parent !== null && !$parent->getChildren()->contains($this)) {
        $parent->addChild($this);
    }

    return $this;
}
```

Assign the owning property before calling inverse-side methods. Together with identity/collection guards, this stops mutual calls from recursing indefinitely.

The setter must correctly support:

- first assignment;
- transfer from one related entity to another;
- assigning the same object again;
- clearing a nullable association;
- removing the object from the previous inverse collection.

## Collection Methods

For `OneToMany`, the collection is normally the inverse side. Its methods must update the child's owning property:

```php
public function addChild(ChildEntity $child): static
{
    if (!$this->children->contains($child)) {
        $this->children->add($child);
        $child->setParent($this);
    }

    return $this;
}

public function removeChild(ChildEntity $child): static
{
    if ($this->children->removeElement($child) && $child->getParent() === $this) {
        $child->setParent(null);
    }

    return $this;
}
```

Keep the identity check during removal. Removing an object from a stale inverse collection must not clear a newer owning-side assignment.

For `ManyToMany`, first determine which collection is owning. Public methods on either side may call their counterpart, but both need `contains()` or successful `removeElement()` guards so the calls terminate and the owning collection is changed.

## Domain Boundaries

- Prefer domain behavior methods over generic setters when an aggregate protects invariants.
- Do not expose an inverse-side mutator if callers are intentionally required to mutate only through an aggregate root.
- Do not perform database queries, call `persist()`, or call `flush()` from entity association methods.
- Do not add global eager loading or change fetch strategy to solve an in-memory synchronization problem.
- Do not confuse association synchronization with lifecycle ownership: cascade and `orphanRemoval` remain separate mapping decisions.

## Verification

Add focused unit tests that do not require a database when only object-graph consistency is changing. Assert:

- the new inverse collection contains the entity;
- the previous inverse collection no longer contains it after reassignment;
- repeated assignment does not create duplicates;
- nullable removal clears both sides;
- direct calls through every supported public side produce the same final graph.

When mapping annotations/attributes change, also validate Doctrine mapping through the project's Docker CLI service. Pure method synchronization without mapping changes does not require a migration.
