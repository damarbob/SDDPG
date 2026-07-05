# Glossary — Domain Dictionary

> **Canonical definitions for project-specific terms used across SDDPG.**
> If a term is not defined here, it has no authoritative meaning in this project.
>
> Terms are ordered alphabetically. Each entry includes a definition, optional aliases, and cross-references to related terms or source documents.

---

### Advisory Lock

A MySQL `GET_LOCK()` call used for mutual exclusion during page provisioning. The Watcher acquires `GET_LOCK('stardust_page_provision', 10)` before executing DDL, preventing concurrent provisioning attempts from causing table name collisions or metadata lock contention.

**See also:** The Watcher, Page.

---

### Backfill Pump

A CLI command (`bin/stardust backfill`) that iterates over historical `entry_data` records in ascending `id` order and pushes them into the event stream for replication into extension tables. It maintains state via a `backfill_checkpoints` table (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.4), allowing resumability (`--from-id`) and reports throughput metrics to stdout. Used during legacy data migration.

**See also:** Dual-Write, Dead Letter Queue, `backfill_checkpoints`.

---

### Bounded Fetch

The second query in the Two-Query Approach (Query 2). After the Paginated Probe identifies a bounded set of matching IDs, the Bounded Fetch retrieves the full row payloads via a safe `WHERE id IN (...)` clause with any necessary extension table joins. The input set is always capped at `page_size`, guaranteeing constant memory usage.

**See also:** Paginated Probe, Two-Query Approach.

---

### Core Payload Table

The primary transactional storage table for all entries. Physically named `entry_data`. Stores the complete, unindexed JSON payload in a `fields` column alongside system timestamps and tenant/model identifiers. Every entry in StarDust has exactly one row in this table regardless of extension table state.

**Aliases:** `entry_data`.
**See also:** Extension Table, Payload Splitting Engine.

---

### Cursor-Based Pagination

The pagination strategy enforced by StarDust's function API. Instead of `OFFSET`-based pagination (which degrades at depth), queries use `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. The `+1` row determines whether a next page exists. The system never evaluates the total matched set of a query, ensuring constant-time pagination regardless of dataset size.

**See also:** Paginated Probe, Two-Query Approach.

---

### Coercion Failure

The Reconciler outcome when a JSON payload value cannot be coerced into the target slot column's type during retype backfill (e.g., `string → int` on the value `"42abc"`, or any of the four categorically-rejected `int↔datetime` / `numeric↔datetime` cells). The slot column receives `NULL`; the JSON payload at `entry_data.fields` remains the authoritative system of record per [ADR 0013](adrs/0013-json-payload-as-system-of-record.md), so reads fall back to `JSON_EXTRACT` without permanent data loss. Coercion failures are explicitly NOT poison pills (they never quarantine to `stardust_reconciler_dlq`); each emits a `coercion_null` structured-log event for operator audit. The full per-type-pair predicate is normative in [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

**See also:** The Reconciler, Tombstoned Slot, [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

---

### Coercion Matrix

The 4×4 lookup table governing how a JSON payload value converts between the four declared types (`string`, `int`, `numeric`, `datetime`) during retype backfill. Each of the twelve off-diagonal cells pins one conversion's predicate and canonical output form, inheriting the QueryFilter typed-value rules (`blueprints/queryfilter_wire_format.md` §4.5) so ingress validation and migration coercion share a single definition of each type. The **identity diagonal** (`string → string`, `int → int`, …) is a no-op pass-through — same-type retypes such as Model Compaction relocations ride it and cannot fail. Four cells are categorically rejected (`int ↔ datetime`, `numeric ↔ datetime`) because epoch interpretation is caller policy, not engine semantics; callers bridge through a `string` intermediate. An attempted-but-failed cell stores `NULL` and emits `coercion_null` (see Coercion Failure). The matrix applies only to the Reconciler retype-backfill path, never to wire-format input validation. Normative in [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

**Aliases:** Type Coercion Matrix.
**See also:** Coercion Failure, Model Compaction, The Reconciler, [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md), [ADR 0016](adrs/0016-field-type-change-lifecycle.md).

---

### Dead Letter Queue (DLQ)

A holding queue for migration event payloads that failed processing by the dual-write consumer worker. Monitored with alerting thresholds: critical alerts fire if DLQ depth exceeds 100 messages or the oldest message age exceeds 12 hours. Failed messages are replayed via `bin/stardust dlq:replay`, which re-submits using the original `entry_id` partition key to preserve causal ordering.

**See also:** Dual-Write, Backfill Pump.

---

### Desync Flag

A state indicator signaling that an entry's indexed representation in one or more extension tables is inconsistent with the current schema registry expectations. Two scenarios produce desync, both with documented resolution paths:

1. **Row desync** — The extension table row is entirely missing. This occurs during an Exhaustion Fallback: the entry is written to `entry_data` only, and its `entry_id` is enqueued to `stardust_sync_queue`. The Reconciler resolves this by backfilling the missing row once capacity is restored.
2. **Index desync** — A field's `is_filterable` flag was promoted from `false` to `true` while the field already had a populated, unindexed slot. Because `ALTER TABLE` on populated pages is forbidden ([ADR 0012](adrs/0012-immutable-extension-page-ddl.md)), the existing slot cannot acquire an index in place. Resolution: the field undergoes the **filterability-promotion lifecycle** defined in [ADR 0016](adrs/0016-field-type-change-lifecycle.md) — the unindexed slot is severed and tombstoned, a new indexed slot of the same type is assigned, and the Reconciler backfills it from the JSON payload. While the new slot is `backfilling`, reads fall back to `JSON_EXTRACT` and filters are rejected (the field is unmapped per the Schema Registry contract in [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md)). Once promoted to `ready`, indexed filtering resumes.

**See also:** Exhaustion Fallback, The Reconciler, `is_filterable`, Page, [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md).

---

### Driver/Adapter Pattern

The extensibility architecture allowing StarDust's read path to be backed by different search engines without altering the ingestion or API layers. The default implementation is the MySQL Native Driver. Alternative drivers (e.g., a Meilisearch driver) can be injected via the engine's Config object at construction time ([ADR 0026](adrs/0026-framework-neutral-composer-packaging.md)). All drivers implement the `EntrySearchInterface` contract and are strictly read-only — writes always target MySQL.

**See also:** `EntrySearchInterface`, MySQL Native Driver, Consistency Header.

---

### Dual-Write

The asynchronous event-driven replication strategy used during legacy data migration. The API synchronous path writes mutations only to the legacy Virtual Column system, then emits a domain event (`EntryCreated`, `EntryUpdated_v2`, `EntryDeleted`) to a message queue using `entry_id` as the partition key. A background consumer worker processes these events and executes idempotent upserts against the new extension tables. Synchronous atomic dual-writes are strictly forbidden.

**See also:** Virtual Column Method, Dead Letter Queue, Backfill Pump.

---

### `entry_data`

The physical MySQL table name for the Core Payload Table.

**See also:** Core Payload Table.

---

### `EntrySearchInterface`

A PHP interface defining the search driver contract for StarDust's read path. Method signatures cover: filtered listing with cursor pagination, single-entry retrieval, and capability introspection (e.g., `supportsFuzzySearch(): bool`). If a caller requests a capability the active driver does not support, the function API rejects the call with a typed exception.

**See also:** Driver/Adapter Pattern, MySQL Native Driver.

---

### `entry_slots_page_X`

The physical MySQL table name pattern for Extension Tables. `X` is a monotonically increasing page number assigned by the Watcher at provisioning time (e.g., `entry_slots_page_1`, `entry_slots_page_2`).

**See also:** Extension Table, Page.

---

### Entry

A single data record belonging to a model within a tenant. Represented as one row in `entry_data` and optionally one row in each relevant `entry_slots_page_X` table. Identified by `entry_id` (aliased as `id` on `entry_data`).

**See also:** Model, Tenant, Core Payload Table.

---

### Exhaustion Fallback

The graceful degradation behavior triggered when all extension table slot capacity reaches 100% and no pages have available capacity. During exhaustion, the write path: (1) writes the full JSON payload to `entry_data`, (2) skips the extension table write entirely, and (3) enqueues the `entry_id` to `stardust_sync_queue`. This ensures high-throughput ingestion never blocks or drops writes, even when the Watcher has failed to provision new pages.

**See also:** Desync Flag, The Watcher, The Reconciler, `stardust_sync_queue`.

---

### Extension Table

A 1:1 table (physically named `entry_slots_page_X`) that stores explicitly indexed fields extracted from an entry's JSON payload. Each extension table contains typed slot columns (`i_str_01`...`i_str_25`, `i_int_01`...`i_int_15`, `i_num_01`...`i_num_10`, `i_dt_01`...`i_dt_10`) and a foreign key to `entry_data` with `ON DELETE CASCADE`. New extension tables are dynamically provisioned by the Watcher when slot capacity runs low.

**Aliases:** Extension Page (informal).
**See also:** Page, Slot, Vertical Schema Partitioning, `entry_slots_page_X`.

---

### Feature Flag (Rollback / Dual-Write)

Two decoupled feature flags used during legacy data migration. The **Read Feature Flag** controls which schema serves read traffic (legacy Virtual Columns or new extension tables). The **Dual-Write Feature Flag** controls whether the event producer emits domain events for replication. These flags are intentionally independent: if reads are rolled back to legacy, the event producer must remain active to prevent data divergence in the extension tables.

**See also:** Dual-Write, Virtual Column Method.

---

### `is_filterable`

A boolean metadata flag in the schema registry, set at model-field registration time. When `true`, a composite B-tree index `(tenant_id, slot_column)` is included in the extension table DDL at page provisioning time. When `false`, the slot is used for discrete retrieval only — any function-API attempt to filter on that field is rejected with a typed exception before the database is touched.

**See also:** Slot, Index Provisioning Policy, Pre-Flight Rejection.

---

### Index Provisioning Policy

The deterministic, schema-driven rules governing which extension table slots receive B-tree indexes. Indexing decisions are tied to the `is_filterable` metadata flag: only slots mapped to fields with `is_filterable = true` are indexed at page creation time. This ensures index provisioning is auditable and never ad-hoc.

**See also:** `is_filterable`, Pre-Flight Rejection, Extension Table, Model-Affine Slot Reservation.

---

### Model

A user-defined data structure (schema) within a tenant, identified by `model_id`. A model defines the set of fields, their types, and their slot mappings in extension tables. All entries belong to exactly one model.

**See also:** Entry, Tenant, Schema Registry.

---

### Model Compaction

The operator-initiated operation that cures Spread: it relocates a fragmented Model's live filterable Slots onto a minimal Page set, restoring the few-joins-per-query property the Slot Spread Metric ([ADR 0031](adrs/0031-slot-spread-metric.md)) measures. Mechanically each relocation is a **same-type retype** riding the unmodified field-lifecycle pipeline ([ADR 0016](adrs/0016-field-type-change-lifecycle.md)) through the coercion matrix's identity diagonal — the only new machinery is a registry-only planner that picks target pages, a page-pinned slot reservation (pin-or-fail; compaction never defers), and a CLI (`bin/stardust compact:model`) that orchestrates relocations **sequentially by default**, so at most one field at a time has filters rejected while its new slot backfills (reads fall back to the JSON payload throughout). Crash recovery is re-run: already-relocated fields are no-ops, so the operation converges idempotently. Never scheduled, never automatic — the operator pays the relocation cost per model, exactly where the metric justifies it. Specified by [ADR 0033](adrs/0033-operator-initiated-model-compaction.md).

**Aliases:** Compaction.
**See also:** Spread, Model-Affine Slot Reservation, Page, Slot, Tombstoned Slot, [ADR 0033](adrs/0033-operator-initiated-model-compaction.md), [ADR 0031](adrs/0031-slot-spread-metric.md), [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [`runbooks/maintaining_low_spread.md`](runbooks/maintaining_low_spread.md).

---

### Model-Affine Slot Reservation

The slot-reservation policy that biases the slot reserver toward free Slots on Pages that already host a live slot of the **same Model**, falling back to the global-oldest-free order when no affine slot of the required type family exists. It is a bias, not an allocation: a model never reserves or owns whole pages, and an affine page's free slots stay available to every other model and tenant. The policy reduces Spread at its source — keeping a model's filterable slots co-located so filtered reads touch fewer pages — without sacrificing slot density, because it only reorders candidates within the existing free-slot pool and never provisions a page the Watcher would not otherwise create. Affinity is forward-only prevention: it keeps fresh and incrementally-grown models compact, but it does not converge models that are already spread (that is operator-initiated compaction) and cannot beat per-family slot ceilings. Specified by [ADR 0032](adrs/0032-model-affine-slot-reservation.md).

**Aliases:** Slot Affinity.
**See also:** Spread, Model Compaction, Slot, Page, Index Provisioning Policy, [ADR 0032](adrs/0032-model-affine-slot-reservation.md), [ADR 0012](adrs/0012-immutable-extension-page-ddl.md).

---

### MySQL Native Driver

The default implementation of `EntrySearchInterface`. Executes queries directly against MySQL using the strict indexing rules defined in the Index Provisioning Policy. Reports `consistencyModel(): "strong"` (callers translate this into their own consumer-facing surface). Rejects filters on non-indexed fields with a typed exception.

**See also:** `EntrySearchInterface`, Driver/Adapter Pattern, Pre-Flight Rejection.

---

### Page

A single numbered instance of an Extension Table, identified by its numeric suffix (e.g., `entry_slots_page_1`). The term "page" emphasizes the **provisioning lifecycle**: pages are created by the Watcher when global slot capacity drops below the configured threshold, and `ALTER TABLE` on populated pages is strictly forbidden. Slot capacity is tracked at the page level.

**See also:** Extension Table, The Watcher, Slot, Spread.

---

### Paginated Probe

The first query in the Two-Query Approach (Query 1). Executes the filter condition as a standalone query selecting only `id` using covering indexes, bounded by cursor logic: `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. The `+1` determines if a next page exists. This query maintains a constant, tiny memory footprint regardless of total matched rows.

**See also:** Bounded Fetch, Two-Query Approach, Cursor-Based Pagination.

---

### Payload Splitting Engine

The write-path component responsible for separating an entry's data into two destinations: the complete JSON payload goes to `entry_data.fields`, while explicitly indexed fields are extracted and written to the appropriate slot columns in the entry's extension table row.

**See also:** Core Payload Table, Extension Table, Slot.

---

### Pre-Flight Rejection

The function API's strict enforcement mechanism for unindexed filter attempts. If a caller requests a filter or sort on a field lacking `is_filterable = true` in the schema registry, the call is immediately aborted at the API boundary with a typed exception before the database is ever touched. This replaces the earlier "Scanned Row Circuit Breaker" concept.

**See also:** `is_filterable`, Index Provisioning Policy, ~~Scanned Row Circuit Breaker~~.

---

### Scanned Row Circuit Breaker _(deprecated)_

A previously proposed runtime mechanism for bounding query execution by tracking scanned rows during query evaluation. This concept has been superseded by Pre-Flight Rejection — strict schema-level enforcement that categorically prevents unindexed queries at the API contract level, eliminating the need for runtime row-count monitoring.

**Replaced by:** Pre-Flight Rejection. See [ADR 0014](adrs/0014-schema-level-safety-over-runtime-circuit-breaking.md).

---

### Schema Registry

The database-resident metadata catalog that tracks extension table pages, slot assignments, field-to-slot mappings, and `is_filterable` flags. It serves as the sole coordination point between the Watcher, the Reconciler, and the ingestion path — no direct IPC exists between these components. The ingestion path reads the registry (with a configurable cache TTL) to determine which page and slot to write indexed fields to.

**See also:** The Watcher, The Reconciler, `is_filterable`, Page.

---

### Shadow Traffic

An optional pre-cutover validation step during legacy data migration (gate 3 of the Three-Gate Protocol). A small percentage of read traffic (e.g., 5%) is routed to the new extension table schema while the majority continues hitting the legacy system. Operators verify P99 latency parity and zero correctness mismatches before proceeding to full cutover.

**See also:** Three-Gate Protocol, Feature Flag.

---

### Slot

A typed column within an extension table (e.g., `i_str_01`, `i_int_15`, `i_num_03`, `i_dt_07`). Each slot has a fixed data type (`VARCHAR`, `BIGINT`, `DOUBLE`, or `DATETIME`) and is mapped to a specific model field via the schema registry. Slots may or may not be indexed depending on the field's `is_filterable` flag. The total number of available slots across all pages determines global capacity.

**See also:** Extension Table, Page, `is_filterable`, Schema Registry, Spread.

---

### Slot Squatting

The capacity exhaustion anti-pattern where fields that are no longer filterable (or entirely deleted) continue to occupy physical column slots in extension tables. The Liberator daemon prevents this by reclaiming and nullifying these slots.

**See also:** The Liberator, Slot, Spread.

---

### Soft Deletion

The temporal deletion strategy used by StarDust. Entries are never physically removed via `DELETE`; instead, the `deleted_at` column on `entry_data` is set to the deletion timestamp. The composite index `(tenant_id, deleted_at, created_at)` supports efficient queries that exclude soft-deleted records.

**See also:** Entry, Core Payload Table.

---

### Spread

The number of distinct extension Pages a single Model's live filterable Slots occupy. Because the query compiler emits one `INNER JOIN entry_slots_page_X` per distinct page a filtered query references, spread is a direct constant-factor cost on filtered reads: a model whose filterable fields are scattered across three pages pays two extra index range-scans versus the same model packed onto one. Spread is an emergent consequence of Immutable Extension Page DDL ([ADR 0012](adrs/0012-immutable-extension-page-ddl.md)) combined with the global-oldest slot-reservation order — incremental field growth and relocations (retype / filterability promotion) land a model's slots on whichever page had a free slot. The advisory **Slot Spread Metric** ([ADR 0031](adrs/0031-slot-spread-metric.md)) measures it per `(tenant, model)` as **excess pages** = pages occupied − theoretical minimum (the fewest pages the model's filterable fields could occupy given per-family slot ceilings), emitting `spread_sampled` (every sample) and `high_spread_model` (when excess crosses a configurable threshold) on `source: registry`. The metric never blocks or rewrites anything; remediation is operator-initiated. Prevention is Model-Affine Slot Reservation; the cure for already-spread models is Model Compaction ([ADR 0033](adrs/0033-operator-initiated-model-compaction.md)).

**Aliases:** Slot Spread, Excess Pages.
**See also:** Page, Slot, Model, Model-Affine Slot Reservation, Model Compaction, The Watcher, [ADR 0031](adrs/0031-slot-spread-metric.md), [ADR 0012](adrs/0012-immutable-extension-page-ddl.md).

---

### `stardust_export_jobs`

The physical MySQL table name for the Chronicler's async export job queue (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.2). Rows transition through `pending → processing → completed | failed`; Chronicler workers claim pending rows via `SELECT ... FOR UPDATE SKIP LOCKED` ordered by per-tenant round-robin position. The 24-hour artifact TTL, 5 GB artifact cap, and ≤ 3 active jobs per tenant are policy from [ADR 0010](adrs/0010-asynchronous-exports.md); daemon-side claim and GC semantics live in [`blueprints/chronicler_daemon.md`](blueprints/chronicler_daemon.md).

**See also:** The Chronicler, Cursor-Based Pagination, Two-Query Approach.

---

### `stardust_reconciler_dlq`

The physical MySQL table name for the Reconciler's per-row poison-pill quarantine (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.3). One row per quarantined entry, distinguished by a `source` discriminator (`sync_queue` or `bulk_import`) so the two Reconciler workloads share one operator surface. Replay is operator-initiated only (`bin/stardust reconciler:dlq:replay`); there is no automatic retry and no automatic TTL. Fully specified by [ADR 0018](adrs/0018-reconciler-poison-pill-semantics.md). Distinct from the migration **Dead Letter Queue (DLQ)** above, which holds dual-write replication failures rather than indexed-materialization failures.

**See also:** The Reconciler, Dead Letter Queue, [ADR 0018](adrs/0018-reconciler-poison-pill-semantics.md).

---

### `stardust_schema_version`

The physical MySQL table name for the registry version counter (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.1). A single-row singleton holding a monotonically increasing `version` column. Every transaction that mutates coordination-relevant registry state — page provisioning, slot-status transitions, field metadata changes — increments `version` in the same transaction as the underlying mutation. Read on every API write path as the cache-invalidation token defined by [ADR 0015](adrs/0015-database-as-sole-daemon-coordination-point.md).

**See also:** Schema Registry, [ADR 0015](adrs/0015-database-as-sole-daemon-coordination-point.md), [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md).

---

### `stardust_sync_queue`

The physical MySQL table name for the ephemeral operations queue. A tiny, dedicated table exclusively for queuing writes that failed due to extension capacity exhaustion. Rows are claimed by the Reconciler via `SELECT ... FOR UPDATE SKIP LOCKED` and deleted after successful backfill. The presence of an `entry_id` in this table is the implicit signal for row-level desync.

**See also:** Exhaustion Fallback, The Reconciler, Desync Flag.

---

### Strict Projection Rule

The architectural directive stating that extension tables are treated strictly as temporal index materializations designed for fast retrieval. The true and authoritative "system of record" always remains the full JSON payload stored within `entry_data.fields`.

**See also:** Extension Table, Core Payload Table.

---

### Tenant

The top-level data isolation boundary in StarDust, identified by `tenant_id`. All queries enforce `tenant_id` matching across `entry_data` and extension table `INNER JOIN` conditions. A tenant's data is completely invisible to other tenants at the query level.

**See also:** Entry, Model.

---

### The Chronicler

An independent background PHP daemon responsible exclusively for materializing export jobs. It claims pending jobs from the exports queue via `SELECT ... FOR UPDATE SKIP LOCKED`, pages through the database using the same Cursor-Based Pagination and Two-Query Approach used by the synchronous read path, and streams the output to a local file on disk (CSV or JSON). Each database operation remains bounded by `page_size`, ensuring the transactional database is never subjected to unbounded queries during export materialization. The Chronicler is independent of the Watcher, Reconciler, and Liberator — it does not participate in daemon-to-daemon coordination (ADR `0015`). It reads `stardust_fields` once per claimed job to derive the CSV header column list (a field-name catalog lookup, not coordination); beyond that one read it never touches the schema registry's slot/page state. How export jobs are _submitted_ (HTTP endpoint, request shape, polling) is the caller's domain.

**See also:** Cursor-Based Pagination, Two-Query Approach, [`blueprints/chronicler_daemon.md`](blueprints/chronicler_daemon.md).

---

### The Liberator

An independent background PHP daemon responsible exclusively for sweeping dead or demoted slots to prevent Slot Squatting. It monitors the schema registry for tombstoned slots and reclaims them using chunked DML nullification without locking tables, ultimately marking them as safely `free` for future indexing needs.

**See also:** Slot Squatting, Tombstoned Slot, Schema Registry.

---

### The Reconciler

An independent, multi-worker background PHP CLI daemon (`bin/stardust reconciler`) responsible for draining the `stardust_sync_queue` and backfilling entries into extension tables. It claims queue rows via `SELECT ... FOR UPDATE SKIP LOCKED` (enabling horizontal scaling), reads the authoritative `entry_data.fields` payload at upsert time (never a stale snapshot), and writes using `INSERT ... ON DUPLICATE KEY UPDATE`. Processes in configurable chunks with inter-chunk delay to prevent write spikes during recovery.

**See also:** The Watcher, Exhaustion Fallback, `stardust_sync_queue`.

---

### The Watcher

A singleton background PHP CLI daemon (`bin/stardust watcher`) responsible for monitoring global slot consumption across all extension tables and provisioning new pages when available capacity drops below the configured threshold (default: 20%). It employs advisory locking, empty-table-only DDL, and atomic registry updates to prevent metadata lock contention. The Watcher's registry update is the sole signal consumed by the Reconciler — no direct notification channel exists.

**See also:** The Reconciler, Page, Advisory Lock, Schema Registry, Spread.

---

### Three-Gate Protocol

The sequential cutover validation process during legacy data migration. Cutover proceeds through three quantifiable gates: (1) **Stream Drain** — consumer group lag holds at `0` for ≥15 minutes; (2) **Data Parity** — random-sample dual-read of ≥10,000 entries yields 100% byte-identical match; (3) **Shadow Traffic** _(optional)_ — a small percentage of reads are routed to the new schema to verify latency and correctness.

**See also:** Dual-Write, Shadow Traffic, Feature Flag.

---

### Tombstoned Slot

The transitional state of an evicted slot. When a field loses its filterable status, its associated slot is severed in the schema registry and marked as "tombstoned". To prevent data bleeding, it cannot be mapped to a new field until The Liberator successfully processes and nullifies all residual data across the respective tenant partition.

**See also:** The Liberator, Slot Squatting, Schema Registry.

---

### Two-Query Approach

The bounded query execution strategy that prevents disk spillage and uncontrolled memory growth on cross-page queries. Consists of two steps: (1) the Paginated Probe (Query 1) selects only `id` values using covering indexes with cursor-based bounds; (2) the Bounded Fetch (Query 2) retrieves full row payloads for the bounded ID set via `WHERE id IN (...)`. This ensures InnoDB never materializes unbounded intermediate result sets.

**Aliases:** Deterministic Late Row Lookups.
**See also:** Paginated Probe, Bounded Fetch, Cursor-Based Pagination.

---

### Vertical Schema Partitioning

The overarching architectural strategy of StarDust. Instead of storing all data in a single wide table (the legacy Virtual Column Method), entries are split across a Core Payload Table (`entry_data`) for the complete JSON payload and 1:1 Extension Tables (`entry_slots_page_X`) for explicitly indexed fields. This separation allows independent scaling of storage and indexing concerns.

**See also:** Extension Table, Core Payload Table, Virtual Column Method.

---

### Virtual Column Method

The legacy single-table architecture being migrated away from. In this approach, all indexed fields were represented as generated virtual columns on a single table, leading to schema rigidity and performance degradation at scale. StarDust's migration to Vertical Schema Partitioning replaces this approach.

**See also:** Vertical Schema Partitioning, Dual-Write, Legacy Data Migration.
