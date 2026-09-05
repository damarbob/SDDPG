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

| ADR                                                                                                                      | Status    | Summary                                                                                                          |
| :----------------------------------------------------------------------------------------------------------------------- | :-------- | :--------------------------------------------------------------------------------------------------------------- |
| [`0000-use-adrs.md`](0000-use-adrs.md)                                                                                   | Accepted  | Decision to use Architecture Decision Records.                                                                   |
| [`0001-extension-tables-over-eav.md`](0001-extension-tables-over-eav.md)                                                 | Accepted  | Chooses vertical extension tables over EAV, sparse-table, and JSON-only storage.                                 |
| [`0002-mysql-native-zero-dependency-core.md`](0002-mysql-native-zero-dependency-core.md)                                 | Accepted  | Establishes MySQL as the standalone core engine with optional external search later.                             |
| [`0003-schema-driven-index-provisioning.md`](0003-schema-driven-index-provisioning.md)                                   | Accepted  | Makes schema metadata the source of truth for which fields are indexed and filterable.                           |
| [`0004-fail-fast-on-unindexed-filters.md`](0004-fail-fast-on-unindexed-filters.md)                                       | Accepted  | Rejects unsupported filters and sorts instead of degrading into unsafe database behavior.                        |
| [`0005-two-query-bounded-read-path.md`](0005-two-query-bounded-read-path.md)                                             | Accepted  | Enforces the paginated probe plus bounded fetch pattern for synchronous reads.                                   |
| [`0006-cursor-based-pagination.md`](0006-cursor-based-pagination.md)                                                     | Accepted  | Mandates cursor-based pagination; forbids OFFSET and COUNT(\*) on the MySQL Native Driver.                       |
| [`0007-write-availability-over-query-completeness.md`](0007-write-availability-over-query-completeness.md)               | Accepted  | Prioritises write availability over indexed query completeness when extension slots are exhausted.               |
| [`0008-singleton-watcher-multi-worker-reconciler.md`](0008-singleton-watcher-multi-worker-reconciler.md)                 | Accepted  | Documents asymmetric scaling: singleton Watcher (DDL safety) vs. multi-worker Reconciler (SKIP LOCKED).          |
| [`0009-tombstone-based-slot-eviction.md`](0009-tombstone-based-slot-eviction.md)                                         | Accepted  | Enforces sever → tombstone → sweep → reclaim lifecycle to prevent data bleeding on slot reuse.                   |
| [`0010-asynchronous-exports.md`](0010-asynchronous-exports.md)                                                           | Relocated | Asynchronous export jobs: per-tenant caps, TTL, format negotiation. Engine-side: `chronicler_daemon.md`.         |
| [`0011-chunked-bulk-ingestion.md`](0011-chunked-bulk-ingestion.md)                                                       | Accepted  | Scopes transactions to individual chunks, trading batch-level atomicity for operational safety.                  |
| [`0012-immutable-extension-page-ddl.md`](0012-immutable-extension-page-ddl.md)                                           | Proposed  | Forbids ALTER TABLE on populated pages; new capacity is added only via new empty tables.                         |
| [`0013-json-payload-as-system-of-record.md`](0013-json-payload-as-system-of-record.md)                                   | Proposed  | Establishes JSON payload as sole authority; extension slots are derived materializations.                        |
| [`0014-schema-level-safety-over-runtime-circuit-breaking.md`](0014-schema-level-safety-over-runtime-circuit-breaking.md) | Proposed  | Rejects runtime circuit breakers; query safety is enforced at the schema level via pre-flight checks.            |
| [`0015-database-as-sole-daemon-coordination-point.md`](0015-database-as-sole-daemon-coordination-point.md)               | Proposed  | Coordinates all daemons exclusively through database state; no message bus or direct IPC.                        |
| [`0016-field-type-change-lifecycle.md`](0016-field-type-change-lifecycle.md)                                             | Proposed  | Defines the retype → tombstone → assign → backfill → promote lifecycle for field type changes.                   |
| [`0017-schema-registry-as-coordination-contract.md`](0017-schema-registry-as-coordination-contract.md)                   | Proposed  | Promotes the schema registry to a first-class contract with normative tables and a closed status enum.           |
| [`0018-reconciler-poison-pill-semantics.md`](0018-reconciler-poison-pill-semantics.md)                                   | Proposed  | Quarantines per-row Reconciler failures into a dedicated DLQ; chunks commit survivors atomically.                |
| [`0019-index-cardinality-policy.md`](0019-index-cardinality-policy.md)                                                   | Proposed  | Asynchronous cardinality sampling emits advisory events; never gates query execution.                            |
| [`0020-structured-logging-mandate.md`](0020-structured-logging-mandate.md)                                               | Proposed  | Mandates NDJSON-to-stdout structured logging with a closed event vocabulary across all runtime sources.          |
| [`0021-search-driver-query-representation.md`](0021-search-driver-query-representation.md)                               | Proposed  | `EntrySearchInterface` accepts a native `QueryFilter` value object; raw DSL passthrough is rejected.             |
| [`0022-search-driver-capability-jurisdiction.md`](0022-search-driver-capability-jurisdiction.md)                         | Proposed  | Filter acceptance is driver-mediated via `supportsFilterOn`; `is_filterable` retains MySQL jurisdiction.         |
| [`0023-minimum-mysql-version.md`](0023-minimum-mysql-version.md)                                                         | Proposed  | Pins the minimum supported database to MySQL 8.0.13; removes the generated-column workaround.                    |
| [`0024-type-coercion-matrix-for-retype-backfill.md`](0024-type-coercion-matrix-for-retype-backfill.md)                   | Proposed  | Pins the per-type-pair coercion predicate for Reconciler retype backfill; mirrors QueryFilter §4.5.              |
| [`0025-chronicler-failure-semantics.md`](0025-chronicler-failure-semantics.md)                                           | Proposed  | Pins Chronicler failure semantics: lease/heartbeat for crashed workers, deadlock budget, bad-row policy.         |
| [`0026-framework-neutral-composer-packaging.md`](0026-framework-neutral-composer-packaging.md)                           | Proposed  | Framework-neutral Composer packaging; CLI via `bin/stardust`; framework adapters are opt-in.                     |
| [`0027-persistent-process-daemon-execution-model.md`](0027-persistent-process-daemon-execution-model.md)                 | Proposed  | Pins persistent-process / container execution as the supported daemon model; defers run-once / cron-driven mode. |
| [`0028-single-document-json-for-import-artifacts.md`](0028-single-document-json-for-import-artifacts.md)                 | Proposed  | Pins async import artifacts as a single JSON document buffered in PHP heap; rejects NDJSON and chunked files.    |
| [`0029-liberator-sweep-omits-tenant-predicate.md`](0029-liberator-sweep-omits-tenant-predicate.md)                       | Accepted  | Liberator tombstone sweep omits the `tenant_id` predicate (single-owner column); refines ADR 0009.               |
| [`0030-string-slot-storage-type-and-index-prefix.md`](0030-string-slot-storage-type-and-index-prefix.md)                 | Accepted  | String slots are `TEXT` with a 766-char prefix index under `ROW_FORMAT=DYNAMIC`, honoring the 4096-char bound.   |
| [`0031-slot-spread-metric.md`](0031-slot-spread-metric.md)                                                               | Accepted  | Advisory per-model spread metric (`excess_pages`); registry-only sampling, never blocks or auto-remediates.      |
| [`0032-model-affine-slot-reservation.md`](0032-model-affine-slot-reservation.md)                                         | Accepted  | Biases slot reservation toward pages hosting the same model's live slots; bias with global-oldest fallback.      |
| [`0033-operator-initiated-model-compaction.md`](0033-operator-initiated-model-compaction.md)                             | Accepted  | Operator-initiated per-model compaction as sequential same-type retypes onto planner-pinned pages.               |
| [`0034-non-filterable-fields-are-json-only.md`](0034-non-filterable-fields-are-json-only.md)                             | Accepted  | Non-filterable fields are JSON-only — never slotted; withdraws ADR 0003's "typed retrieval" allowance.           |
| [`0035-usable-capacity-is-a-satisfiability-test.md`](0035-usable-capacity-is-a-satisfiability-test.md)                   | Proposed  | Watcher provisions on a per-family satisfiability test, not a usable-capacity ratio; refines ADR 0016.           |
| [`0036-entry-payload-keys-are-field-names.md`](0036-entry-payload-keys-are-field-names.md)                               | Proposed  | Payload keys stay field names, not ids; field rename is a chunked backfill bridged by `previous_name`.           |
| [`0037-field-deletion-lifecycle.md`](0037-field-deletion-lifecycle.md)                                                   | Proposed  | Field deletion severs immediately, purges payloads asynchronously, and drops the registry row last.              |
| [`0038-model-deletion-lifecycle.md`](0038-model-deletion-lifecycle.md)                                                   | Proposed  | Model deletion severs immediately, hard-purges entries in chunks, and drops the model row last.                  |
| [`0039-compaction-refuses-to-plan-mid-lifecycle.md`](0039-compaction-refuses-to-plan-mid-lifecycle.md)                   | Proposed  | Compaction refuses to plan while a field of the model is mid-lifecycle; narrows ADR 0033's "resume is re-run".   |
| [`0040-import-manifest-enumerates-chunks.md`](0040-import-manifest-enumerates-chunks.md)                                 | Proposed  | The async import manifest enumerates chunks with id ranges; `rolled_back` is scoped to the synchronous path.     |
| [`0041-sort-ordering-and-the-anchored-cursor.md`](0041-sort-ordering-and-the-anchored-cursor.md)                         | Proposed  | Adds an optional sort to reads; makes ADR 0004's long-vacuous "reject sorts" clause real.                        |
| [`0042-index-headroom-at-page-provisioning.md`](0042-index-headroom-at-page-provisioning.md)                             | Proposed  | Pages are provisioned with `k` indexed columns per family, not demand-sized; narrows ADR 0003.                   |
| [`0043-pages-provision-only-indexed-columns.md`](0043-pages-provision-only-indexed-columns.md)                           | Proposed  | Pages are created with only the slot columns they index, so free inventory means claimable inventory.            |
