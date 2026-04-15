# Architecture Decision Records (ADRs)

> **An immutable log of _why_ key technical decisions were made.**

## Purpose

This directory contains Architecture Decision Records (ADRs) for the StarDust project. ADRs document the context, decisions, and consequences for significant architectural choices (e.g., "Why extension tables over EAV?"). They serve as institutional memory, preventing the team from re-litigating settled debates and providing historical context for future maintainers.

## When to Write an ADR

Write an ADR when a proposed change or decision:

- Introduces new architectural patterns or paradigms.
- Has a massive impact on performance, operability, scalability, or maintainability.
- Replaces or significantly modifies an existing core architectural decision.
- Requires a deliberate trade-off between competing technical goals.

## Conventions

| Rule             | Convention                                                                                                                                                                                                                  |
| :--------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File format**  | Markdown (`.md`)                                                                                                                                                                                                            |
| **File naming**  | `NNNN-short-title.md` (e.g., `0001-extension-tables-over-eav.md`)                                                                                                                                                           |
| **Template**     | [`_template.md`](_template.md) — copy this to start a new ADR                                                                                                                                                               |
| **Immutability** | ADRs are **append-only**. Do not edit the core decision or context of a published ADR. If a decision is changed later, create a new ADR that supersedes the original, and update the status of the old one to `Superseded`. |
| **Status**       | `Proposed` → `Accepted`, `Rejected`, `Deprecated`, `Superseded`                                                                                                                                                             |

## Index

| ADR                                                                                                        | Status   | Summary                                                                                            |
| :--------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------- |
| [`0000-use-adrs.md`](0000-use-adrs.md)                                                                     | Accepted | Decision to use Architecture Decision Records.                                                     |
| [`0001-extension-tables-over-eav.md`](0001-extension-tables-over-eav.md)                                   | Accepted | Chooses vertical extension tables over EAV, sparse-table, and JSON-only storage.                   |
| [`0002-mysql-native-zero-dependency-core.md`](0002-mysql-native-zero-dependency-core.md)                   | Accepted | Establishes MySQL as the standalone core engine with optional external search later.               |
| [`0003-schema-driven-index-provisioning.md`](0003-schema-driven-index-provisioning.md)                     | Accepted | Makes schema metadata the source of truth for which fields are indexed and filterable.             |
| [`0004-fail-fast-on-unindexed-filters.md`](0004-fail-fast-on-unindexed-filters.md)                         | Accepted | Rejects unsupported filters and sorts instead of degrading into unsafe database behavior.          |
| [`0005-two-query-bounded-read-path.md`](0005-two-query-bounded-read-path.md)                               | Accepted | Enforces the paginated probe plus bounded fetch pattern for synchronous reads.                     |
| [`0006-cursor-based-pagination.md`](0006-cursor-based-pagination.md)                                       | Accepted | Mandates cursor-based pagination; forbids OFFSET and COUNT(\*) on the MySQL Native Driver.         |
| [`0007-write-availability-over-query-completeness.md`](0007-write-availability-over-query-completeness.md) | Accepted | Prioritises write availability over indexed query completeness when extension slots are exhausted. |
