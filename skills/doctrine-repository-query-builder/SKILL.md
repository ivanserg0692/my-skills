---
name: doctrine-repository-query-builder
description: Create or refactor Doctrine repository methods in Symfony services using reusable QueryBuilder helper methods. Use when adding findAll/findOne repository queries, reducing duplicated leftJoin/addSelect chains, preloading relations to avoid N+1 queries, sharing relation-loading logic such as addStoreRelations/addSectionsRelation between repositories, or fixing Doctrine pagination with fetch joins on to-many collections.
---

# Doctrine Repository Query Builder

## Workflow

1. Read the target repository and related repositories before editing.
2. Find existing `QueryBuilder` helper methods before adding new join logic.
3. Use Doctrine `QueryBuilder` for repository queries unless the user explicitly asks for raw SQL or DQL.
4. Extract repeated `leftJoin()` / `addSelect()` relation chains into repository helper methods.
5. Reuse helper methods from query methods such as `findAllWith...()` and `findOneWith...()`.
6. Preserve existing filters, sorting, aliases, return types, and business behavior unless the user explicitly requests a change.
7. Validate changed PHP files with `php -l`; when Doctrine mapping may be involved, run `doctrine:schema:validate --skip-sync` through the project Docker CLI service.

## Helper Method Pattern

Create helper methods on the repository that owns the entity relation.

Use this shape:

```php
public function addStoreRelations(
    QueryBuilder $queryBuilder,
    string $elementAlias = 'c',
    string $storeStockAlias = 'storeStock',
    string $storeAlias = 'store',
): QueryBuilder {
    return $queryBuilder
        ->leftJoin(sprintf('%s.storeStocks', $elementAlias), $storeStockAlias)
        ->addSelect($storeStockAlias)
        ->leftJoin(sprintf('%s.store', $storeStockAlias), $storeAlias)
        ->addSelect($storeAlias);
}
```

Rules:

- Accept aliases as parameters so the helper can be reused from another repository.
- Make helper methods public only when they are intentionally reused by another repository or caller.
- Keep helper methods private when they are local implementation details of a single repository.
- Return the same `QueryBuilder` for chaining.
- Use stable, descriptive default aliases.
- Keep helper methods focused on relation loading; do not hide filters or business-specific sorting inside generic helpers.
- Add `use Doctrine\ORM\QueryBuilder;` when helper methods type-hint it.

## Query Method Pattern

Build query methods by composing helpers:

```php
public function findAllWithStoreStocks(): array
{
    $queryBuilder = $this->createQueryBuilder('c');

    $this->addStoreRelations($queryBuilder);
    $this->addSectionsRelation($queryBuilder);

    return $queryBuilder
        ->getQuery()
        ->getResult();
}
```

For single-item methods, add the identifier filter after relation helpers unless existing code requires a different order:

```php
return $queryBuilder
    ->andWhere('c.id = :id')
    ->setParameter('id', $id)
    ->getQuery()
    ->getOneOrNullResult();
```

## Paginated To-Many Fetch Joins

Do not paginate a query that fetch joins one-to-many or many-to-many collections.

Treat this as a problem when a list query combines:

- `Doctrine\ORM\Tools\Pagination\Paginator` or `setFirstResult()` / `setMaxResults()`
- `leftJoin()` / `innerJoin()` plus `addSelect()` for collection relations
- a root entity result that must still serialize or expose those collections without N+1 queries

Use a two-step entity load plus a separate count:

1. Query page ids only.
   - Select only the root entity id.
   - Apply the same filters as the list query.
   - Apply the same root sorting.
   - Use `setFirstResult()` and `setMaxResults()` here.
   - Do not `addSelect()` collection relations.
   - Join only the minimum relations required for filters.
2. Load entities by ids.
   - Return ORM entities, not DTOs, unless the task explicitly asks otherwise.
   - Use `WHERE root.id IN (:ids)`.
   - Fetch join the relations needed by serialization or read logic.
   - Do not paginate this query.
   - Restore the original page order from the input ids after hydration.
3. Count total separately.
   - Use a simple ORM count query.
   - Apply the same filters.
   - Use `COUNT(DISTINCT root.id)` when filters can join to-many relations.
   - Do not fetch join collections.

Keep filters in a private helper when the ids and count queries need identical predicates:

```php
private function applyListFilters(QueryBuilder $queryBuilder, ?int $sectionId, ?bool $active): QueryBuilder
{
    if ($sectionId !== null) {
        $queryBuilder
            ->innerJoin("entity.sections", "filterSection")
            ->andWhere("filterSection.id = :sectionId")
            ->setParameter("sectionId", $sectionId);
    }

    if ($active !== null) {
        $queryBuilder
            ->andWhere("entity.active = :active")
            ->setParameter("active", $active);
    }

    return $queryBuilder;
}
```

Do not change the API or serialization contract while doing this refactor. Preserve the original filters, sorting, page and limit behavior, serializer groups, and returned entity type unless the user explicitly approves a behavior change.

## Cross-Repository Reuse

When repository `A` needs to preload relations that belong to entity repository `B`, inject repository `B` through constructor injection and call its helper.

Example:

```php
public function __construct(
    ManagerRegistry $registry,
    private readonly CatalogElementsRepository $catalogElementsRepository,
) {
    parent::__construct($registry, CatalogSections::class);
}
```

Then:

```php
$queryBuilder = $this->createQueryBuilder('section')
    ->leftJoin('section.catalogElements', 'element')
    ->addSelect('element');

$this->catalogElementsRepository->addStoreRelations($queryBuilder, 'element');
```

## N+1 Guidance

- Preload only relations needed by the requested read model.
- Prefer query-level joins over global eager fetch mapping.
- Do not add self-joins for tree structures unless the user explicitly asks for nested tree hydration.
- For bounded-depth trees, confirm the depth or use an existing nested-set/materialized-path strategy when available.

## Safety

- Do not edit Doctrine migration files for repository-only refactoring.
- Do not change entity mappings, cascade options, orphan removal, or fetch mode as part of repository helper cleanup unless explicitly requested.
- Do not change sorting direction, filter semantics, pagination, or result shape while deduplicating joins.
