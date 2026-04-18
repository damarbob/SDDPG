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

| ADR                                                                                                                      | Status   | Summary                                                                                                 |
| :----------------------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------ |
| [`0000-use-adrs.md`](0000-use-adrs.md)                                                                                   | Accepted | Decision to use Architecture Decision Records.                                                          |
| [`0001-extension-tables-over-eav.md`](0001-extension-tables-over-eav.md)                                                 | Accepted | Chooses vertical extension tables over EAV, sparse-table, and JSON-only storage.                        |
| [`0002-mysql-native-zero-dependency-core.md`](0002-mysql-native-zero-dependency-core.md)                                 | Accepted | Establishes MySQL as the standalone core engine with optional external search later.                    |
| [`0003-schema-driven-index-provisioning.md`](0003-schema-driven-index-provisioning.md)                                   | Accepted | Makes schema metadata the source of truth for which fields are indexed and filterable.                  |
| [`0004-fail-fast-on-unindexed-filters.md`](0004-fail-fast-on-unindexed-filters.md)                                       | Accepted | Rejects unsupported filters and sorts instead of degrading into unsafe database behavior.               |
| [`0005-two-query-bounded-read-path.md`](0005-two-query-bounded-read-path.md)                                             | Accepted | Enforces the paginated probe plus bounded fetch pattern for synchronous reads.                          |
| [`0006-cursor-based-pagination.md`](0006-cursor-based-pagination.md)                                                     | Accepted | Mandates cursor-based pagination; forbids OFFSET and COUNT(\*) on the MySQL Native Driver.              |
| [`0007-write-availability-over-query-completeness.md`](0007-write-availability-over-query-completeness.md)               | Accepted | Prioritises write availability over indexed query completeness when extension slots are exhausted.      |
| [`0008-singleton-watcher-multi-worker-reconciler.md`](0008-singleton-watcher-multi-worker-reconciler.md)                 | Accepted | Documents asymmetric scaling: singleton Watcher (DDL safety) vs. multi-worker Reconciler (SKIP LOCKED). |
| [`0009-tombstone-based-slot-eviction.md`](0009-tombstone-based-slot-eviction.md)                                         | Accepted | Enforces sever → tombstone → sweep → reclaim lifecycle to prevent data bleeding on slot reuse.          |
| [`0010-asynchronous-exports.md`](0010-asynchronous-exports.md)                                                           | Accepted | Provides `/api/exports` (202 Accepted) for unbounded retrieval without relaxing synchronous limits.     |
| [`0011-chunked-bulk-ingestion.md`](0011-chunked-bulk-ingestion.md)                                                       | Accepted | Scopes transactions to individual chunks, trading batch-level atomicity for operational safety.         |
| [`0012-immutable-extension-page-ddl.md`](0012-immutable-extension-page-ddl.md)                                           | Accepted | Forbids ALTER TABLE on populated pages; new capacity is added only via new empty tables.                |
| [`0013-json-payload-as-system-of-record.md`](0013-json-payload-as-system-of-record.md)                                   | Accepted | Establishes JSON payload as sole authority; extension slots are derived materializations.               |
| [`0014-schema-level-safety-over-runtime-circuit-breaking.md`](0014-schema-level-safety-over-runtime-circuit-breaking.md) | Accepted | Rejects runtime circuit breakers; query safety is enforced at the schema level via pre-flight checks.   |
