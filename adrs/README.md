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
| **Immutability** | ADRs are **append-only**. Do not edit the core decision or context of a published ADR. A later ADR leaves a dated note on the older one, as set out in [Notes on an earlier ADR](#notes-on-an-earlier-adr), and only a wholesale replacement changes the older ADR's status, to `Superseded`. |
| **Status**       | `Proposed` → `Accepted`, `Rejected`, `Deprecated`, `Superseded`                                                                                                                                                             |

## Notes on an earlier ADR

A later ADR that extends, narrows, amends or corrects an earlier one leaves a dated note on the earlier one, so that a reader of the old ADR finds the change. The old text is not rewritten. While the earlier ADR is `Proposed` it may be edited in place instead, and an `Accepted` one takes only editorial fixes in place (a typo, a broken link, a sentence reworded without changing any claim). A note is also the right record on a `Proposed` ADR when the later ADR is itself `Proposed`, because the later one may be rejected.

### Choosing the label

Ask what happens to the older claim.

| The older claim is | Label | Example |
| :-- | :-- | :-- |
| Still true, with nothing changed | Note | ADR 0023, noting ADR 0054 |
| Still true, with more built on top | Extended | ADR 0006, by ADR 0041 |
| True only in a smaller scope | Narrowed | ADR 0005, by ADR 0056 |
| Left open or ambiguous, and now pinned down | Clarified | ADR 0009, by ADR 0046 |
| Wrong | Corrected | ADR 0030, by ADR 0041 |
| No longer true as written, because behaviour changed | Amended | ADR 0009, by ADR 0049 |
| Replaced for good | Superseded | ADR 0031, by ADR 0044 |

One note records one change. A later ADR that both extends and narrows an earlier one leaves two notes.

### Writing the note

A note is a blockquote placed directly under the sentence, list item or bullet it concerns. It opens with the bold label, the date and the later ADR, and then says in one or two sentences what holds now. The later ADR owns the reasoning, so the note does not repeat it.

```text
> **Narrowing proposed 2026-11-14 by ADR [`0059`](0059-table-names-take-an-optional-installation-prefix.md):** with a table prefix set, two installations can share one database, so the discriminator would be `DATABASE()` and the prefix together. This item stands until that ADR is accepted, and it keeps applying to an installation with no prefix.
```

The date is the day the note was written. A note that no other ADR prompted leaves out "by ADR", as in `**Corrected 2026-08-17:**`. The later ADR lists the earlier one under Related with what changes in brackets, and its index row says what it narrows, extends or amends.

### While the later ADR is Proposed

| The later ADR is | Label | Wording |
| :-- | :-- | :-- |
| `Proposed` | The noun of the label followed by "proposed", such as Narrowing proposed, Extension proposed, Amendment proposed, Clarification proposed or Supersession proposed | "Would", and a closing sentence saying the item stands until that ADR is accepted |
| `Accepted` | The past tense label | What now holds, with the closing sentence dropped |

Note and Corrected record a fact that is already known, so they have no proposed form. When a `Proposed` ADR is accepted, each of its notes changes from the proposed label to the past tense one, and the closing sentence is dropped.

### Status

Only Superseded changes the status of the earlier ADR, and only when the whole ADR is replaced. The status then reads `Superseded` and the note names the replacing ADR. Every other label leaves the status as it is. `Relocated` is a status value used once, by ADR 0010, for a decision that moved to another repository, and it stays as it is.

### Notes written before this convention

Notes written before 2026-11-14 keep their wording and are not rewritten. For counting or searching them, Scoped reads as Narrowed, Refined as Clarified, Addendum as Note, and a bare "Amended" line inside a bullet as an Amended note. A few notes carry no label, and they are left as they are.

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
| [`0012-immutable-extension-page-ddl.md`](0012-immutable-extension-page-ddl.md)                                           | Accepted  | Forbids ALTER TABLE on populated pages; new capacity is added only via new empty tables.                         |
| [`0013-json-payload-as-system-of-record.md`](0013-json-payload-as-system-of-record.md)                                   | Proposed  | Establishes JSON payload as sole authority; extension slots are derived materializations.                        |
| [`0014-schema-level-safety-over-runtime-circuit-breaking.md`](0014-schema-level-safety-over-runtime-circuit-breaking.md) | Proposed  | Rejects runtime circuit breakers; query safety is enforced at the schema level via pre-flight checks.            |
| [`0015-database-as-sole-daemon-coordination-point.md`](0015-database-as-sole-daemon-coordination-point.md)               | Proposed  | Coordinates all daemons exclusively through database state; no message bus or direct IPC.                        |
| [`0016-field-type-change-lifecycle.md`](0016-field-type-change-lifecycle.md)                                             | Accepted  | Defines the retype → tombstone → assign → backfill → promote lifecycle for field type changes.                   |
| [`0017-schema-registry-as-coordination-contract.md`](0017-schema-registry-as-coordination-contract.md)                   | Proposed  | Promotes the schema registry to a first-class contract with normative tables and a closed status enum.           |
| [`0018-reconciler-poison-pill-semantics.md`](0018-reconciler-poison-pill-semantics.md)                                   | Proposed  | Quarantines per-row Reconciler failures into a dedicated DLQ; chunks commit survivors atomically.                |
| [`0019-index-cardinality-policy.md`](0019-index-cardinality-policy.md)                                                   | Proposed  | Asynchronous cardinality sampling emits advisory events; never gates query execution.                            |
| [`0020-structured-logging-mandate.md`](0020-structured-logging-mandate.md)                                               | Proposed  | Mandates NDJSON-to-stdout structured logging with a closed event vocabulary across all runtime sources.          |
| [`0021-search-driver-query-representation.md`](0021-search-driver-query-representation.md)                               | Proposed  | `EntrySearchInterface` accepts a native `QueryFilter` value object; raw DSL passthrough is rejected.             |
| [`0022-search-driver-capability-jurisdiction.md`](0022-search-driver-capability-jurisdiction.md)                         | Proposed  | Filter acceptance is driver-mediated via `supportsFilterOn`; `is_filterable` retains MySQL jurisdiction.         |
| [`0023-minimum-mysql-version.md`](0023-minimum-mysql-version.md)                                                         | Proposed  | Pins the minimum supported database to MySQL 8.0.13; removes the generated-column workaround.                    |
| [`0024-type-coercion-matrix-for-retype-backfill.md`](0024-type-coercion-matrix-for-retype-backfill.md)                   | Accepted  | Pins the per-type-pair coercion predicate for Reconciler retype backfill; mirrors QueryFilter §4.5.              |
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
| [`0044-theoretical-minimum-pages-from-real-capacity.md`](0044-theoretical-minimum-pages-from-real-capacity.md)           | Proposed  | Derives `theoretical_min_pages` from real, heterogeneous per-page capacity instead of fixed per-family constants; keeps ADR 0031's advisory and ADR 0033's planner agreeing by construction. |
| [`0045-tombstoning-resets-the-sweep-annotations.md`](0045-tombstoning-resets-the-sweep-annotations.md)                   | Proposed  | Tombstoning now resets `sweep_cursor_id` and `sweep_gap_count`; closes the bleed where a recycled slot's second sweep resumed from its prior occupant's cursor and returned to `free` still holding its values. |
| [`0046-a-gapped-sweep-skips-the-chunk-and-does-not-reclaim.md`](0046-a-gapped-sweep-skips-the-chunk-and-does-not-reclaim.md) | Proposed  | Fixes the Liberator's gap path to advance by rows, not id arithmetic, and to skip an abandoned chunk rather than reclaim before it is verified empty, per ADR 0009. |
| [`0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md`](0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md) | Proposed  | Pins the export resume anchor to the artifact's verified byte offset, not a trusted `last_cursor`; supersedes ADR 0025 Commitment 1's resume half. |
| [`0048-bounded-combined-tick-for-cron-driven-hosting.md`](0048-bounded-combined-tick-for-cron-driven-hosting.md)         | Proposed  | Ships ADR 0027's deferred `--once` mode for three of four daemons: one bounded, budget-limited tick over one connection, for cron-only MySQL-8 hosting. Chronicler stays excluded pending a cooperative yield. |
| [`0049-multi-worker-liberator-excluded-per-page.md`](0049-multi-worker-liberator-excluded-per-page.md)                   | Proposed  | Multi-workers the Liberator via page-table-granularity `GET_LOCK` exclusion; narrows ADR 0009's singleton and rejects `SKIP LOCKED` at slot granularity. |
| [`0050-chronicler-cooperative-yield-at-a-chunk-boundary.md`](0050-chronicler-cooperative-yield-at-a-chunk-boundary.md)   | Proposed  | Lets an export yield back to `pending` at a committed chunk boundary with its ADR 0047 anchor intact, so `tick --exports` can bound a large job. |
| [`0051-the-disk-gate-is-a-write-probe-not-a-free-space-ratio.md`](0051-the-disk-gate-is-a-write-probe-not-a-free-space-ratio.md) | Proposed  | Replaces the pre-claim free-space ratio with a write probe, after measuring that a per-uid quota is invisible to `statvfs` while a project quota is not. |
| [`0052-the-advisory-schedule-is-persisted-and-fleet-wide.md`](0052-the-advisory-schedule-is-persisted-and-fleet-wide.md) | Proposed  | Moves the Watcher's advisory schedule into a singleton table so it survives a per-cron-run process; makes the sample fleet-wide rather than per-daemon. |
| [`0053-advisory-lock-names-are-qualified-per-installation.md`](0053-advisory-lock-names-are-qualified-per-installation.md) | Proposed  | Qualifies both `GET_LOCK` names with a per-installation suffix, after measuring that lock names are server-scoped and two installations on one shared mysqld excluded each other. |
| [`0054-mariadb-10-11-support-floor.md`](0054-mariadb-10-11-support-floor.md)                                             | Proposed  | Makes MariaDB 10.11+ a supported second engine with `utf8mb4_unicode_520_nopad_ci` and a generated-column substitute for the functional unique index; excludes 10.6 and accepts one supplementary-plane sort divergence. |
| [`0055-target-engine-is-detected-not-declared.md`](0055-target-engine-is-detected-not-declared.md)                       | Accepted  | Detects the target engine from the live connection instead of configuring it, failing closed below each engine's floor; rejects a `Config`-declared target and capability probing. |
| [`0056-filtered-default-order-reads-drive-from-a-full-width-equality-leaf.md`](0056-filtered-default-order-reads-drive-from-a-full-width-equality-leaf.md) | Accepted  | Drives a filtered default-order read from a full-width equality leaf under a pinned join order, bounding discovery for integer, numeric and datetime equality; narrows ADR 0005.   |
| [`0057-string-slots-are-indexed-through-a-generated-key.md`](0057-string-slots-are-indexed-through-a-generated-key.md) | Accepted  | Replaces the 766-character prefix index on pages built from now on with a stored generated key (a 32-byte weight-string head plus a digest), so a text equality is exact and can start a bounded read. It extends ADR 0056. |
| [`0058-a-filterable-field-on-an-empty-model-is-born-with-its-slot.md`](0058-a-filterable-field-on-an-empty-model-is-born-with-its-slot.md) | Proposed  | Reserves an indexed slot at registration for a filterable field on a model with no entries, so it is queryable at once; a populated model keeps the lazy path. It would narrow a clarification in ADR 0007. |
| [`0059-table-names-take-an-optional-installation-prefix.md`](0059-table-names-take-an-optional-installation-prefix.md) | Proposed  | Adds an optional `Config::$tablePrefix` that every table, named constraint, lock name and daemon file carries, so StarDust can share a database with other applications and with a second installation. It would narrow a premise of ADR 0053. |
