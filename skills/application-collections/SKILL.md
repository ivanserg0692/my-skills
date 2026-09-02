---
name: application-collections
description: Use when reviewing, designing, or refactoring application/domain code where arrays, lists, DTO lists, or Doctrine entity collections are repeatedly mapped, filtered, grouped, validated, indexed, or passed between layers. Helps decide when to keep a plain array, create a named DTO/application collection, use an entity collection, or avoid premature generic abstractions.
---

# Application Collections

## Purpose

Use this skill to decide whether a repeated array/list pattern should remain a plain array or become a named application collection. Prefer this for application-layer and domain-layer refactoring discussions, especially when code starts accumulating `map*`, `filter*`, `group*`, `validate*`, or `indexBy*` helper methods around the same data set.

The goal is not to remove every array. The goal is to give a business-relevant set of values a name when that set has stable meaning, behavior, or invariants inside a use case.

## Core Rule

Create a collection when an array has stopped being just a list.

A list has usually become a concept when the same set of elements is repeatedly:

- validated before use;
- mapped into several downstream payloads;
- grouped or indexed by the same key;
- filtered according to scenario rules;
- passed through several services as a single participant in the use case;
- coupled to ordering, uniqueness, quantity, totals, or other invariants.

Keep a plain array when the operation is local, one-off, and the list has no clear scenario name.

## Refactoring Signal

A common path to a collection is this smell:

```php
mapCartItemsToCheckoutItems();
mapPricesByProductId();
mapDeductionsByProductId();
mapToDeductStocksItems();
validateCartItems();
```

These methods may be correct, but they often describe technical transformations instead of the model. If they orbit the same data set, ask what object is missing.

Prefer moving toward language like:

```php
$items = CheckoutCartItems::fromCartItems($cart->getItems());

$items->getProductIds();
$items->toDeductStocksItems();
$orderItemSource->getPriceForProduct($productId);
$orderItemSource->getDeductionForProduct($productId);
```

The improvement is not syntax. The improvement is that the code now says what the set means in the scenario.

## Choosing The Shape

Use a plain array when:

- the list is created and consumed inside one method;
- there are no invariants to protect;
- there is no useful name beyond `items`, `rows`, or `ids`;
- a collection would only hide a simple `foreach`;
- the abstraction would make readers jump to another file for no extra meaning.

Use a DTO/application collection when:

- the list is a named participant in a use case;
- validation belongs to application-side input preparation;
- several steps need different projections of the same elements;
- the collection can expose scenario methods such as `getProductIds()`, `toDeductStocksItems()`, `getTotalQuantity()`, or `isEmpty()`;
- using raw arrays makes the use-case service know too much about element structure.

Use a Doctrine entity collection when:

- the behavior is about the persisted aggregate or entity relationship;
- the method expresses domain state owned by the entity;
- Doctrine lifecycle, cascade, orphan removal, or relation consistency is part of the concern.

Do not put application checkout/export/import/request behavior into Doctrine collections just because the source data came from the database. Prefer a DTO/application collection for scenario-specific projections.

## Naming Heuristics

Name the collection after the scenario meaning, not the storage type.

Good names:

```php
CheckoutCartItems
OrderImportRows
PricedOrderItems
NotificationRecipients
```

Weaker names:

```php
CartItemCollection
ItemsArrayWrapper
EntityList
DataCollection
```

Prefer method names that describe scenario projections:

```php
fromCartItems()
toCheckoutItems()
getProductIds()
toDeductStocksItems()
getByProductId()
```

Avoid generic mapper names when a domain name is available:

```php
mapItems()
mapData()
groupThings()
processArray()
```

Use `get` prefix for read accessors when the project follows PHP/Symfony getter conventions. Use `to*` for conversion/export to another representation. Use `from*` for named construction from another model.

## Generic Abstractions

Do not start with a reusable generic collection. First create the concrete collection that the use case needs.

Only introduce a generic helper when several concrete collections need the same behavior and the shared abstraction would reduce real duplication without erasing domain language.

Avoid this too early:

```php
GroupedCollection::by($items, fn ($item) => $item->id);
```

Prefer this until repetition is proven:

```php
$items->getByProductId($productId);
```

The second form keeps the use case readable and leaves room to enforce invariants.

## Checkout Example

In checkout code, `CartItem[]` from the database is storage-oriented data. Once those items are validated and prepared for checkout, the set has a new application meaning.

A `CheckoutCartItems` collection is justified when it:

- validates product ids and quantities before remote calls;
- exposes `getProductIds()` for price fetching;
- exposes `toDeductStocksItems()` for inventory deduction;
- preserves ordering for order item creation;
- keeps `OrderApiService` focused on orchestration instead of array plumbing.

This collection should not know how to call gRPC, persist orders, or calculate every financial field unless that behavior is truly about the item set itself.

## Review Checklist

When reviewing code, ask:

1. What is the business name of this list at this point in the scenario?
2. Are multiple `map`, `filter`, `group`, or `validate` methods orbiting the same list?
3. Are invariants repeated or trusted implicitly?
4. Is the service doing low-level array plumbing instead of orchestrating the use case?
5. Would a named collection reduce coupling to raw DTO/entity structure?
6. Would the collection add real language, or only hide a loop?
7. Does the behavior belong to persisted entity state, or to an application scenario?

Create the collection only when the answers show stable meaning and repeated behavior.
