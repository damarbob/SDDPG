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

| ADR                                                                                                                      | Status   | Summary                                                                                                  |
| :----------------------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------- |
| [`0000-use-adrs.md`](0000-use-adrs.md)                                                                                   | Accepted | Decision to use Architecture Decision Records.                                                           |
| [`0001-extension-tables-over-eav.md`](0001-extension-tables-over-eav.md)                                                 | Proposed | Chooses vertical extension tables over EAV, sparse-table, and JSON-only storage.                         |
| [`0002-mysql-native-zero-dependency-core.md`](0002-mysql-native-zero-dependency-core.md)                                 | Proposed | Establishes MySQL as the standalone core engine with optional external search later.                     |
| [`0003-schema-driven-index-provisioning.md`](0003-schema-driven-index-provisioning.md)                                   | Proposed | Makes schema metadata the source of truth for which fields are indexed and filterable.                   |
| [`0004-fail-fast-on-unindexed-filters.md`](0004-fail-fast-on-unindexed-filters.md)                                       | Proposed | Rejects unsupported filters and sorts instead of degrading into unsafe database behavior.                |
| [`0005-two-query-bounded-read-path.md`](0005-two-query-bounded-read-path.md)                                             | Proposed | Enforces the paginated probe plus bounded fetch pattern for synchronous reads.                           |
| [`0006-cursor-based-pagination.md`](0006-cursor-based-pagination.md)                                                     | Proposed | Mandates cursor-based pagination; forbids OFFSET and COUNT(\*) on the MySQL Native Driver.               |
| [`0007-write-availability-over-query-completeness.md`](0007-write-availability-over-query-completeness.md)               | Proposed | Prioritises write availability over indexed query completeness when extension slots are exhausted.       |
| [`0008-singleton-watcher-multi-worker-reconciler.md`](0008-singleton-watcher-multi-worker-reconciler.md)                 | Proposed | Documents asymmetric scaling: singleton Watcher (DDL safety) vs. multi-worker Reconciler (SKIP LOCKED).  |
| [`0009-tombstone-based-slot-eviction.md`](0009-tombstone-based-slot-eviction.md)                                         | Proposed | Enforces sever → tombstone → sweep → reclaim lifecycle to prevent data bleeding on slot reuse.           |
| [`0010-asynchronous-exports.md`](0010-asynchronous-exports.md)                                                           | Relocated | Asynchronous export jobs: per-tenant caps, TTL, format negotiation. Engine-side: `chronicler_daemon.md`. |
| [`0011-chunked-bulk-ingestion.md`](0011-chunked-bulk-ingestion.md)                                                       | Proposed | Scopes transactions to individual chunks, trading batch-level atomicity for operational safety.          |
| [`0012-immutable-extension-page-ddl.md`](0012-immutable-extension-page-ddl.md)                                           | Proposed | Forbids ALTER TABLE on populated pages; new capacity is added only via new empty tables.                 |
| [`0013-json-payload-as-system-of-record.md`](0013-json-payload-as-system-of-record.md)                                   | Proposed | Establishes JSON payload as sole authority; extension slots are derived materializations.                |
| [`0014-schema-level-safety-over-runtime-circuit-breaking.md`](0014-schema-level-safety-over-runtime-circuit-breaking.md) | Proposed | Rejects runtime circuit breakers; query safety is enforced at the schema level via pre-flight checks.    |
| [`0015-database-as-sole-daemon-coordination-point.md`](0015-database-as-sole-daemon-coordination-point.md)               | Proposed | Coordinates all daemons exclusively through database state; no message bus or direct IPC.                |
| [`0016-field-type-change-lifecycle.md`](0016-field-type-change-lifecycle.md)                                             | Proposed | Defines the retype → tombstone → assign → backfill → promote lifecycle for field type changes.           |
| [`0017-schema-registry-as-coordination-contract.md`](0017-schema-registry-as-coordination-contract.md)                   | Proposed | Promotes the schema registry to a first-class contract with normative tables and a closed status enum.   |
| [`0018-reconciler-poison-pill-semantics.md`](0018-reconciler-poison-pill-semantics.md)                                   | Proposed | Quarantines per-row Reconciler failures into a dedicated DLQ; chunks commit survivors atomically.        |
| [`0019-index-cardinality-policy.md`](0019-index-cardinality-policy.md)                                                   | Proposed | Asynchronous cardinality sampling emits advisory events; never gates query execution.                    |
| [`0020-structured-logging-mandate.md`](0020-structured-logging-mandate.md)                                               | Proposed | Mandates NDJSON-to-stdout structured logging with a closed event vocabulary across all runtime sources.  |
| [`0021-search-driver-query-representation.md`](0021-search-driver-query-representation.md)                               | Proposed | `EntrySearchInterface` accepts a native `QueryFilter` value object; raw DSL passthrough is rejected.     |
| [`0022-search-driver-capability-jurisdiction.md`](0022-search-driver-capability-jurisdiction.md)                         | Proposed | Filter acceptance is driver-mediated via `supportsFilterOn`; `is_filterable` retains MySQL jurisdiction. |
| [`0023-minimum-mysql-version.md`](0023-minimum-mysql-version.md)                                                         | Proposed | Pins the minimum supported database to MySQL 8.0.13; removes the generated-column workaround.            |
