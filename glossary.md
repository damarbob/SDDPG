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

A CLI command (`php spark stardust:backfill`) that iterates over historical `entry_data` records in ascending `id` order and pushes them into the event stream for replication into extension tables. It maintains state via a `backfill_checkpoints` table, allowing resumability (`--from-id`) and reports throughput metrics to stdout. Used during legacy data migration.

**See also:** Dual-Write, Dead Letter Queue.

---

### Bounded Fetch

The second query in the Two-Query Approach (Query 2). After the Paginated Probe identifies a bounded set of matching IDs, the Bounded Fetch retrieves the full row payloads via a safe `WHERE id IN (...)` clause with any necessary extension table joins. The input set is always capped at `page_size`, guaranteeing constant memory usage.

**See also:** Paginated Probe, Two-Query Approach.

---

### Consistency Header

An HTTP response header (`X-StarDust-Consistency`) included in every search response. Its value reflects the active search driver's consistency model: `strong` for the MySQL Native Driver (reads from the transactional store) or `eventual` for drivers backed by asynchronous indexes. Enables consumers to make informed caching and retry decisions.

**See also:** `EntrySearchInterface`, Driver/Adapter Pattern.

---

### Core Payload Table

The primary transactional storage table for all entries. Physically named `entry_data`. Stores the complete, unindexed JSON payload in a `fields` column alongside system timestamps and tenant/model identifiers. Every entry in StarDust has exactly one row in this table regardless of extension table state.

**Aliases:** `entry_data`.
**See also:** Extension Table, Payload Splitting Engine.

---

### Cursor-Based Pagination

The pagination strategy mandated by the StarDust API. Instead of `OFFSET`-based pagination (which degrades at depth), queries use `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. The `+1` row determines whether a next page exists. The system never evaluates the total matched set of a query, ensuring constant-time pagination regardless of dataset size.

**See also:** Paginated Probe, Two-Query Approach.

---

### Dead Letter Queue (DLQ)

A holding queue for migration event payloads that failed processing by the dual-write consumer worker. Monitored with alerting thresholds: critical alerts fire if DLQ depth exceeds 100 messages or the oldest message age exceeds 12 hours. Failed messages are replayed via `php spark stardust:dlq:replay`, which re-submits using the original `entry_id` partition key to preserve causal ordering.

**See also:** Dual-Write, Backfill Pump.

---

### Desync Flag

A state indicator signaling that an entry's indexed representation in one or more extension tables is inconsistent with the current schema registry expectations. Two scenarios produce desync:

1. **Row desync** — The extension table row is entirely missing. This occurs during an Exhaustion Fallback: the entry is written to `entry_data` only, and its `entry_id` is enqueued to `stardust_sync_queue`. The Reconciler resolves this by backfilling the missing row once capacity is restored.
2. **Index desync** — The extension table row exists, but it resides on a page that was provisioned *before* a field's `is_filterable` flag was changed to `true`. The required B-tree index is absent on that page. Because `ALTER TABLE` on populated pages is forbidden, the index cannot be retroactively added. *(Resolution strategy is an open architectural gap — see note below.)*

> [!WARNING]
> Index desync (scenario 2) has no documented resolution mechanism. This is a candidate for a future Architecture Decision Record.

**See also:** Exhaustion Fallback, The Reconciler, `is_filterable`, Page.

---

### Driver/Adapter Pattern

The extensibility architecture allowing StarDust's read path to be backed by different search engines without altering the ingestion or API layers. The default implementation is the MySQL Native Driver. Alternative drivers (e.g., a Meilisearch driver) can be injected via CI4 service configuration. All drivers implement the `EntrySearchInterface` contract and are strictly read-only — writes always target MySQL.

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

A PHP interface defining the search driver contract for StarDust's read path. Method signatures cover: filtered listing with cursor pagination, single-entry retrieval, and capability introspection (e.g., `supportsFuzzySearch(): bool`). If a consumer requests a capability the active driver does not support, the API returns `400 Bad Request`.

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

A boolean metadata flag in the schema registry, set at model-field registration time. When `true`, a composite B-tree index `(tenant_id, slot_column)` is included in the extension table DDL at page provisioning time. When `false`, the slot is used for discrete retrieval only — any API attempt to filter on that field is rejected with `400 Bad Request` before the database is touched.

**See also:** Slot, Index Provisioning Policy, Pre-Flight Rejection.

---

### Index Provisioning Policy

The deterministic, schema-driven rules governing which extension table slots receive B-tree indexes. Indexing decisions are tied to the `is_filterable` metadata flag: only slots mapped to fields with `is_filterable = true` are indexed at page creation time. This ensures index provisioning is auditable and never ad-hoc.

**See also:** `is_filterable`, Pre-Flight Rejection, Extension Table.

---

### Model

A user-defined data structure (schema) within a tenant, identified by `model_id`. A model defines the set of fields, their types, and their slot mappings in extension tables. All entries belong to exactly one model.

**See also:** Entry, Tenant, Schema Registry.

---

### MySQL Native Driver

The default implementation of `EntrySearchInterface`. Executes queries directly against MySQL using the strict indexing rules defined in the Index Provisioning Policy. Reports `X-StarDust-Consistency: strong`. Rejects filters on non-indexed fields with `400 Bad Request`.

**See also:** `EntrySearchInterface`, Driver/Adapter Pattern, Pre-Flight Rejection.

---

### Page

A single numbered instance of an Extension Table, identified by its numeric suffix (e.g., `entry_slots_page_1`). The term "page" emphasizes the **provisioning lifecycle**: pages are created by the Watcher when global slot capacity drops below the configured threshold, and `ALTER TABLE` on populated pages is strictly forbidden. Slot capacity is tracked at the page level.

**See also:** Extension Table, The Watcher, Slot.

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

The API's strict enforcement mechanism for unindexed filter attempts. If a consumer requests a filter or sort on a field lacking `is_filterable = true` in the schema registry, the CodeIgniter 4 request is immediately aborted with `400 Bad Request` before the database is ever touched. This replaces the earlier "Scanned Row Circuit Breaker" concept.

**See also:** `is_filterable`, Index Provisioning Policy, ~~Scanned Row Circuit Breaker~~.

---

### Scanned Row Circuit Breaker *(deprecated)*

A previously proposed runtime mechanism for bounding query execution by tracking scanned rows during query evaluation. This concept has been superseded by Pre-Flight Rejection — strict schema-level enforcement that categorically prevents unindexed queries at the API contract level, eliminating the need for runtime row-count monitoring.

> [!NOTE]
> The reference to "configurable circuit breaker" in Architecture Blueprint §1 is a stale artifact and should be updated in a future cleanup pass.

**Replaced by:** Pre-Flight Rejection.

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

**See also:** Extension Table, Page, `is_filterable`, Schema Registry.

---

### Soft Deletion

The temporal deletion strategy used by StarDust. Entries are never physically removed via `DELETE`; instead, the `deleted_at` column on `entry_data` is set to the deletion timestamp. The composite index `(tenant_id, deleted_at, created_at)` supports efficient queries that exclude soft-deleted records.

**See also:** Entry, Core Payload Table.

---

### `stardust_sync_queue`

The physical MySQL table name for the ephemeral operations queue. A tiny, dedicated table exclusively for queuing writes that failed due to extension capacity exhaustion. Rows are claimed by the Reconciler via `SELECT ... FOR UPDATE SKIP LOCKED` and deleted after successful backfill. The presence of an `entry_id` in this table is the implicit signal for row-level desync.

**See also:** Exhaustion Fallback, The Reconciler, Desync Flag.

---

### Tenant

The top-level data isolation boundary in StarDust, identified by `tenant_id`. All queries enforce `tenant_id` matching across `entry_data` and extension table `INNER JOIN` conditions. A tenant's data is completely invisible to other tenants at the query level.

**See also:** Entry, Model.

---

### The Reconciler

An independent, multi-worker background PHP CLI daemon (`php spark stardust:reconciler`) responsible for draining the `stardust_sync_queue` and backfilling entries into extension tables. It claims queue rows via `SELECT ... FOR UPDATE SKIP LOCKED` (enabling horizontal scaling), reads the authoritative `entry_data.fields` payload at upsert time (never a stale snapshot), and writes using `INSERT ... ON DUPLICATE KEY UPDATE`. Processes in configurable chunks with inter-chunk delay to prevent write spikes during recovery.

**See also:** The Watcher, Exhaustion Fallback, `stardust_sync_queue`.

---

### The Watcher

A singleton background PHP CLI daemon (`php spark stardust:watcher`) responsible for monitoring global slot consumption across all extension tables and provisioning new pages when available capacity drops below the configured threshold (default: 20%). It employs advisory locking, empty-table-only DDL, and atomic registry updates to prevent metadata lock contention. The Watcher's registry update is the sole signal consumed by the Reconciler — no direct notification channel exists.

**See also:** The Reconciler, Page, Advisory Lock, Schema Registry.

---

### Three-Gate Protocol

The sequential cutover validation process during legacy data migration. Cutover proceeds through three quantifiable gates: (1) **Stream Drain** — consumer group lag holds at `0` for ≥15 minutes; (2) **Data Parity** — random-sample dual-read of ≥10,000 entries yields 100% byte-identical match; (3) **Shadow Traffic** *(optional)* — a small percentage of reads are routed to the new schema to verify latency and correctness.

**See also:** Dual-Write, Shadow Traffic, Feature Flag.

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
