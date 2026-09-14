---
name: encapsulate-infrastructure-data
description: Use when reviewing or implementing integrations where Elasticsearch, SQL, HTTP, or another infrastructure exposes structured request/response arrays that could leak across application layers.
---

# Encapsulate Infrastructure Data

## Purpose

Keep external protocol representations behind the class that owns the integration. This applies to Elasticsearch query bodies and responses, SQL parameter/result shapes, HTTP client payloads, and similar infrastructure data.

The goal is not to prohibit arrays. The goal is to prevent an untyped external representation from becoming an implicit contract between unrelated layers.

## Core Rule

Do not expose raw infrastructure arrays as a read/write protocol between layers when a focused builder, adapter, DTO, value object, or response model can hide the representation.

The class that owns the external protocol should:

- build and mutate request structures;
- translate external response structures into application models;
- validate or normalize protocol-specific fields;
- hide field names, nesting, pagination markers, and transport options from callers.

Callers should express intent through methods such as `buildPageQuery()`, `nextPageQuery()`, `fromDocument()`, or `toResponse()`, rather than assigning protocol keys directly.

## Review Signals

Treat these patterns as a signal to introduce or extend an encapsulation boundary:

- `$body["from"] = ...` or `$body["search_after"] = ...` outside an Elasticsearch query builder;
- repeated `[$source]["field"]` access outside the adapter/DTO responsible for document conversion;
- controller or application code depending on transport-specific keys such as `hits`, `pit`, `relation`, or `shards`;
- several callers duplicating the same query fragments, null handling, pagination flags, or response normalization;
- an `array<string, mixed>` returned from infrastructure and passed through multiple layers unchanged.

## Preferred Design

Keep raw arrays local to a narrow infrastructure class. Expose the smallest intentional API required by its caller:

- query builders expose named operations for filters, sorting, pagination, and aggregations;
- Elasticsearch/HTTP adapters expose application DTOs or typed result objects;
- response DTOs provide explicit factories when they are intentionally the boundary for a document shape;
- pagination state is represented by methods or a dedicated object instead of caller-side mutation of request arrays.

When new protocol fields are needed, extend the owning boundary first. Do not spread another string-key lookup to every caller.

## Boundaries and Exceptions

Plain arrays remain appropriate when they are local implementation details, simple DTO collections, or a framework-required payload at the final transport call. A boundary class may return an array when the framework client requires it, but callers should not modify that array or rely on its internal shape.

Do not introduce wrappers merely to rename a one-off local list. Apply the rule where the array represents an external protocol, crosses a layer boundary, or carries stable structure and behavior.

Preserve existing behavior and public contracts. For legacy code, prefer a focused adapter or incremental boundary around the changed path instead of broad unrelated rewrites.

## Implementation Checklist

1. Identify which class owns the external protocol.
2. Move construction and mutation of protocol arrays into that class.
3. Convert response data at the boundary into DTOs/value objects or expose narrow accessors.
4. Remove raw key access from application and controller layers.
5. Add tests for the boundary's request shape and for the caller's behavior, without making callers assert implementation details unnecessarily.
6. Check that adding a future filter, sort, aggregation, or response field requires extending one boundary rather than changing every consumer.
