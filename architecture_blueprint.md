# StarDust Core Architecture Blueprint

This document defines the core architecture for StarDust, which utilizes a **Vertical Schema Partitioning (Extension Tables)** strategy.

## 1. Core Architectural Pillars

- **Architectural Foundation**: The system utilizes a 1:1 extension table strategy (`entry_data` + `entry_slots_page_X`), optimized strictly as a high-throughput ingestion and discrete retrieval engine.
- **Strict Resource Bounding**: Ensure MySQL query execution is tightly bounded via pre-flight schema validation and strictly enforced schema indexing, preventing disk spillage of temporary tables without relying on external 3rd-party search infrastructure.
- **Stable Consumer Contract**: Provide explicit, deterministic API endpoints. The system will gracefully reject unoptimized queries (e.g., filtering on unindexed fields) with a `400 Bad Request` rather than silently degrading database performance.
- **Extensible Search Architecture (Adapter Pattern)**: Ship the core system as a purely standalone, zero-dependency relational engine to minimize the infrastructural barrier to entry. Future integration with dedicated search engines (e.g., Meilisearch, OpenSearch) is facilitated through an open Driver/Adapter interface at the CodeIgniter 4 application layer.
- **Operational Resilience**: Guarantee data integrity under degraded states (e.g., slot exhaustion) through a robust fallback queue (`stardust_sync_queue`) and two independent background daemons — **The Watcher** (capacity provisioning) and **The Reconciler** (queue draining) — operating as isolated failure domains.

### 1.1 Operating Environment

StarDust targets MySQL 8.0.13 or newer (Percona Server 8.0.13+ is supported on the same basis). Older MySQL versions and MariaDB are explicitly out of scope: the schema registry's atomicity invariants rely on partial unique indexes, the registry diagnostic queries rely on common table expressions, and the operator runbooks for cardinality advisories rely on `EXPLAIN ANALYZE` — all of which require the 8.0.13 floor. Implementations may not target older versions; no compatibility shim or generated-column workaround is supported.

---

## 2. Technical Specifications & Storage Strategy

MySQL is treated strictly as a transactional store, not a catch-all search engine.

- **`entry_data` (Core Payload Table)**
  - `id` (BIGINT, Primary Key)
  - `tenant_id` (BIGINT, Index for tenant isolation)
  - `model_id` (INT)
  - `created_at`, `updated_at`, `deleted_at` (DATETIME, System timestamps)
  - `fields` (JSON, Complete unindexed payload)
  - _Indexes_: `(tenant_id, model_id)`, `(tenant_id, deleted_at, created_at)`
- **`stardust_sync_queue` (Ephemeral Operations Queue)**
  - `id` (BIGINT, Primary Key)
  - `entry_id` (BIGINT)
  - `created_at` (DATETIME)
  - _Role_: A tiny, dedicated table exclusively for queuing writes that failed due to extension capacity constraints. Polled by `SELECT ... FOR UPDATE SKIP LOCKED`.
- **`entry_slots_page_X` (1:1 Extension Tables)**
  - `entry_id` (BIGINT, Primary Key / Foreign Key `ON DELETE CASCADE`)
  - `tenant_id` (BIGINT)
  - `i_str_01`...`i_str_25` (VARCHAR), `i_int_01`...`i_int_15` (BIGINT), `i_num_01`...`i_num_10` (DOUBLE), `i_dt_01`...`i_dt_10` (DATETIME)
  - **⚠️ Selective Indexing**: Indexes are governed by the Index Provisioning Policy (§2.2).

### 2.1 Automated Page Provisioning & Exhaustion Fallback

Three independent background PHP daemons manage extension page lifecycle and data recovery. They share no direct IPC; the schema registry (database) is the sole coordination point between them. The registry's concrete shape — tables, slot status enum, atomicity boundaries — is defined in §2.3.

- **⚠️ Exhaustion Fallback (Resilience)**: If the Watcher fails and slot capacity reaches 100%, high-throughput ingestion **must not block or drop writes**. The payload splitting engine gracefully degrades: it writes the full JSON payload into the `entry_data` table, skips writing to the `entry_slots_page_X` table, and enqueues the entry ID into the `stardust_sync_queue` table. The Reconciler will backfill these entries once capacity is restored.

#### 2.1.1 The Watcher (Capacity Monitor & Page Provisioner)

A **singleton** background daemon responsible exclusively for monitoring global slot consumption across all `entry_slots_page_X` tables and provisioning new pages when available capacity drops below 20%.

- **Execution Profile**: Lazy polling interval (configurable, default: `60s`). The Watcher is not latency-sensitive — it runs on a leisurely schedule.
- **Safeguards**: Employs Empty-Table-Only DDL, Advisory Locking (`GET_LOCK('stardust_page_provision', 10)`), and Atomic Registry Updates to prevent metadata lock contention. `ALTER TABLE` on populated pages is strictly forbidden.
- **Provisioning Signal**: On successful page creation, the Watcher atomically updates the schema registry. This registry update is the sole signal consumed by the Reconciler — no direct notification is required.

#### 2.1.2 The Reconciler (Queue Drain & Data Backfill)

An independent background daemon responsible exclusively for draining the `stardust_sync_queue` and backfilling entries into extension tables.

- **Queue Polling**: Continuously polls `stardust_sync_queue` utilizing `SELECT ... FOR UPDATE SKIP LOCKED` to concurrently claim tasks without metadata lock contention on the main ingestion thread.
- **Capacity Check**: On each poll cycle, the Reconciler queries the schema registry for available slot capacity. If capacity exists, it backfills entries using `INSERT ... ON DUPLICATE KEY UPDATE` and deletes the corresponding queue rows. If no capacity exists, it sleeps and retries.
- **Backpressure**: Processes in configurable chunks (default: `500`) with a configurable delay between chunks to prevent write spikes against extension tables during recovery.
- **Observability**: Reports throughput metrics to stdout (rows processed / elapsed time).
- **Multi-Worker Safe**: `SKIP LOCKED` provides natural row-level mutual exclusion, allowing horizontal scaling during recovery bursts.

#### 2.1.3 The Liberator (Slot Eviction & Capacity Reclamation)

An independent background daemon responsible exclusively for sweeping "dead" slots to prevent capacity exhaustion ("Slot Squatting"). Operating under the **Strict Projection Rule**, extension tables are treated purely as index materializations; the true system of record remains the `fields` JSON payload.

- **Eviction Triggers**: If a field is entirely deleted or its `is_filterable` flag is demoted to `false`, its physical mapping (e.g., `i_str_01`) is immediately severed in the Schema Registry. The API instantly falls back to `JSON_EXTRACT` for subsequent queries (see §3).
- **Tombstoning**: To prevent data bleeding (where a newly mapped field accidentally inherits old data), the severed slot is marked `tombstoned` in the registry. It cannot be mapped to a new field for that model.
- **Background Sweeping**: The Liberator lazily processes tombstoned slots without locking tables, executing chunked DML nullification (`UPDATE entry_slots_page_X SET i_str_XX = NULL WHERE tenant_id = ? AND id > ? LIMIT 500`).
- **Capacity Reclamation**: Once The Liberator confirms a slot is 100% nullified for a tenant/model partition, it updates the registry to mark the slot `free`, making it safely available for a new indexable field.

#### 2.1.4 Coordination & Concurrency Constraints

- **The Watcher is a strict singleton.** Multiple Watcher instances racing to provision the same page will cause DDL conflicts despite advisory locking (lock timeouts, duplicate table names). Enforce via PID file or OS-level process locking.
- **The Reconciler supports multiple workers.** `SELECT ... FOR UPDATE SKIP LOCKED` guarantees row-level mutual exclusion across concurrent workers.
- **Failure Isolation**: Any daemon can crash independently without affecting the others. If the Watcher dies, the Reconciler continues draining into existing pages with remaining capacity. If the Reconciler dies, the Watcher continues provisioning — the queue grows, but writes never block because the exhaustion fallback (above) keeps ingestion operational. If the Liberator dies, dead slots simply stay `tombstoned` and squat on capacity until it recovers. Ingestion and synchronous retrieval are never blocked by daemon failures.

#### 2.1.5 Slot Assignment Lifecycle

When a new model field is registered in the schema registry, the system assigns it a physical slot column in two steps:

1. **Type resolution**: The field's declared type (`string`, `int`, `numeric`, `datetime`) maps to the corresponding column family (`i_str_XX`, `i_int_XX`, `i_num_XX`, `i_dt_XX`).
2. **Slot reservation**: The registry scans existing extension pages for a free slot of the matching type. The first available slot is atomically reserved in the registry and marked `assigned` to the field. No DDL is executed — slot reservation is a pure registry write. If no free slot of the required type exists on any provisioned page, the field is left unmapped until the Watcher provisions a new page and the reservation is retried.

Once assigned, the payload splitting engine extracts the field's value from each incoming entry's `fields` JSON payload and writes it directly to the reserved slot column. For entries written during a capacity gap (and therefore queued via the exhaustion fallback), the Reconciler performs the same extraction via `JSON_EXTRACT` during its backfill pass.

The assigned slot is the physical target for all slot-based reads, index filters, and the Liberator's sweep — it does not change unless the field is evicted or its type changes.

#### 2.1.6 Field Type Change Lifecycle

When a model field's declared type is changed (e.g., `string → int`), its existing slot is physically incompatible with the new type. The system follows a **retype → tombstone → assign → backfill → promote** lifecycle. There is no dedicated `retyping` slot status — the transition is expressed as coordinated state changes across two `stardust_slot_assignments` rows (the old slot and the new slot), committed together where possible.

1. **Retype (atomic registry transaction)**: In one commit, the field's `declared_type` is updated, the old slot flips `status: assigned → tombstoned` with `field_id → NULL` (entering the standard sweep-nullify-reclaim lifecycle under the Liberator — §2.1.3), and — if a free slot of the target type is available on any existing page — a new slot flips `status: free → backfilling` with `field_id` set to this field. The field's live slot is now `backfilling`, so slot-based reads and index-based filtering are both suppressed regardless of `is_filterable`; all reads fall back to `JSON_EXTRACT`. **No new page is provisioned** by this step.
2. **Deferred assignment (only if no free slot of the target type existed)**: The old slot is tombstoned, but no new slot was assigned in step 1. The field is temporarily unmapped and reads fall back to `JSON_EXTRACT`. When the Watcher independently detects low capacity and provisions a new page (§2.1.1), the standard slot assignment process (§2.1.5) reserves a slot of the target type and transitions it `free → backfilling` with `field_id` set.
3. **Backfill via Reconciler**: A backfill record is enqueued. The Reconciler processes it in configurable chunks using `JSON_EXTRACT` + type coercion. Entries where the value cannot be coerced (e.g., a non-numeric string being retyped to `int`) store `NULL` in the new slot and fall back to `JSON_EXTRACT` on retrieval. **Filterability remains suppressed** (slot status = `backfilling`) until backfill completes.
4. **Promote**: Once the Reconciler confirms all rows in the tenant/model partition are processed, the slot status advances to `ready`. If `is_filterable = true`, the composite index on the new slot becomes active for query routing.

### 2.2 Index Provisioning Policy

Indexing decisions are **schema-driven**, governed by the `is_filterable` metadata flag set at model-field registration time.

- If `is_filterable = true` → a composite B-tree index `(tenant_id, slot_column)` is included in the page DDL at provisioning time.
- If `is_filterable = false` → the slot is used for discrete retrieval only. **Filters against non-filterable fields are strictly forbidden.** To maintain low latency, MySQL will never evaluate `JSON_EXTRACT` inside a `WHERE` clause. If a consumer attempts to filter on an unindexed field, the API will strictly reject the request with an **HTTP 400 Bad Request**. No fallback or suggestions for in-memory bulk filtering will be provided, forcing the consumer to request proper schema provisioning if the filter is a genuine business requirement.
- This ensures index provisioning is a deterministic, auditable decision tied to the schema registry.

### 2.3 Schema Registry (Field-to-Slot Mapping)

Slot columns (`i_str_01`, `i_int_15`, etc.) are generically named and carry no domain meaning. The **schema registry** is the database-resident metadata catalog that records which physical slot each model field occupies, the lifecycle state of that slot, and the `is_filterable` flag driving index routing. It is the sole coordination surface shared by the payload splitting engine, the read path, and the three daemons. No daemon signals another except by writing registry state; there is no message bus, pub/sub channel, or direct IPC between them.

The registry is normative, not implementation detail. Its full DDL and normative table definitions are specified in the Schema Reference.

- **Four tables**: `stardust_models`, `stardust_fields`, `stardust_pages`, `stardust_slot_assignments`.
- **Closed slot status enum**: `free | assigned | tombstoned | backfilling | ready`. Every transition in §2.1.3, §2.1.5, and §2.1.6 moves a `stardust_slot_assignments` row between these states. A retype is expressed as coordinated transitions across two rows (old slot tombstoned + new slot assigned to `backfilling`), not as a single intermediate state.
- **Uniqueness at the database level**: `UNIQUE (page_id, slot_column)` prevents two fields from racing onto the same physical column; a partial unique on `field_id` filtered to live statuses (`assigned`, `backfilling`, `ready`) prevents a field from occupying two slots at once. The old slot of an in-flight retype is `tombstoned` with `field_id = NULL` and does not block the new slot's assignment.
- **Atomic transitions**: each of the following must commit inside a single registry transaction:
  - **Sever + tombstone**: `status: assigned → tombstoned` and `field_id → NULL` in one commit. Readers never observe a slot still marked "assigned" but no longer mapped.
  - **Retype + tombstone old + assign new**: `declared_type` update, old slot `assigned → tombstoned` (`field_id → NULL`), and new slot `free → backfilling` (`field_id` set) all commit together. If no free slot of the target type exists, the old slot is still tombstoned in that commit and the new-slot assignment is deferred to §2.1.5 once capacity is restored.
  - **Sweep completion + slot free**: the Liberator's final nullification batch and `tombstoned → free` commit together. A slot cannot be reassigned until the sweep-confirm is durable.
  - **Page provisioning + slot inventory insert**: the Watcher's `CREATE TABLE` is followed immediately by a single transaction inserting the `stardust_pages` row and all new `stardust_slot_assignments` rows (`status = 'free'`). The page is never visible to the Reconciler before its slot inventory is registered.

On every write, the payload splitting engine looks up the live `(page, slot_column)` for each field via `stardust_fields` → `stardust_slot_assignments`. On every read, the same lookup determines whether `?select=fieldName` resolves to an `INNER JOIN` on the assigned slot column, or falls back to `JSON_EXTRACT(fields, '$.fieldName')` from `entry_data`. The `fields` JSON column is the authoritative system of record; slot columns are index materializations derived from it.

---

## 3. The API Contract & Consumer Abstraction

The API must never leak the physical database design or silently shift consistency models without explicit headers.

- **Driver-Based Architecture**:
  - To keep the maintenance layer minimal, StarDust uses a standard `EntrySearchInterface`.
  - The default driver is the **MySQL Native Driver**, which enforces the strict indexing rules defined in Section 2.2.
  - If future enterprise clients require arbitrary reporting or full-text search, developers can inject a 3rd-party driver (e.g., `MeilisearchDriver`) without altering the core ingestion engine.
- **API Purity & Predictability**:
  - Consumers use `/api/entries` for standard synchronous data retrieval.
  - To minimize the maintenance layer, relying on external 3rd-party services (like OpenSearch) is strongly discouraged. As a standalone engine, StarDust does not perform transparent proxying or dynamic shifts to eventual consistency.
  - To support scale without breaking the consumer contract, `/api/entries` mandates **Cursor-Based Pagination**. The system never evaluates the total matched set of a query; it only evaluates the requested page (e.g., `LIMIT 50`). This ensures valid business queries are never arbitrarily rejected with 422 errors simply because the result set grew over time.
  - If a query references a field lacking `is_filterable = true` in the schema registry, the API rejects the request with **HTTP 400 Bad Request** before any database interaction (see Section 4). There is no runtime row-count tracking or query-time abort mechanism — safety is enforced exclusively at the schema-validation layer.
- **Seamless Field Selection**: Consumers request fields logically using standard query parameters (e.g., `?select=firstName,jobId`). The system dynamically determines if a field requires an `INNER JOIN` against an extension table or a `JSON_EXTRACT(fields, '$.propertyName')` from the core payload on retrieval (select only, never for filtering), automatically merging the final representation. This guarantees that demoted (`is_filterable = false`) or unindexed fields are seamlessly supported via the core payload without squatting on finite physical schema slots.

---

## 4. The Read Path: Bounded Query Execution

The CodeIgniter 4 query builder ensures InnoDB is protected from runaway materialize operations without punishing consumers for large datasets.

- **Tenant Isolation**: Secure all `INNER JOIN` conditions by enforcing `tenant_id` matches across all pages.
- **Deterministic Late Row Lookups (Two-Query Approach)**: Cross-page queries and complex filters must never be executed as a single, potentially unbound `INNER JOIN`. To guarantee bounded execution and prevent disk spillage, the executor enforces a strict two-step process:
  1. **Query 1 (Paginated Probe)**: Execute the filter condition as a standalone query selecting _only_ `id` using covering indexes, bounding the query using standard cursor logic: `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. This determines if a next page exists while maintaining a constant, tiny memory footprint regardless of total matched rows.
  2. **Query 2 (Bounded Fetch)**: Use the resulting array of IDs (up to `page_size`) to fetch the full row payloads via a safe `WHERE id IN (...)` clause with any necessary joins.

### 4.1 Strict Schema Enforcement

The read path actively avoids the illusion of dynamically tracking "scanned rows" via internal metrics before query execution. Database safety is instead guaranteed entirely through rigorous schema-level enforcement by the default `MySQL Native Driver`.

- **Predictable Execution (No Fallbacks)**: The system does not attempt to catch low-cardinality index scans or "poorly formed queries" dynamically at runtime. Instead, the driver categorically prevents unindexed filtering at the API contract level (as defined in Section 2.2).
- **Index Responsibility**: The framework delegates the responsibility of query performance—ensuring high index cardinality—entirely to the schema designer. The native driver relies strictly on the `LIMIT {page_size} + 1` bound applied to a covering index in the Paginated Probe (Query 1) to prevent catastrophic application memory spikes. A slow index scan caused by a low-selectivity filter is treated as an architectural configuration failure, not a runtime exception.
- **Immediate Pre-Flight Rejection**: Any API attempt to filter or sort on a field lacking an explicit `is_filterable = true` registry flag immediately aborts the CodeIgniter 4 request with an **HTTP 400 Bad Request** before the database is ever touched.

### 4.2 Asynchronous Exports (Massive Data Retrieval)

To maintain the strict synchronous page limits on `/api/entries` without abandoning consumers who genuinely need massive datasets, StarDust utilizes a dedicated export pattern.

- **The `/api/exports` Endpoint**: A consumer posts a large query filter to `/api/exports`. The API accepts the payload, validates the query, and responds immediately with a `202 Accepted` and an export Job ID.
- **Background Materialization**: **The Chronicler** — an independent background PHP daemon (separate from the Watcher, Reconciler, and Liberator) — picks up the export job. It seamlessly pages through the database using the same Cursor-Based Pagination described above (Query 1 and Query 2 in a loop) and writes the output locally to a streaming CSV or JSON file on disk.
- **Result Delivery**: Once complete, the consumer can poll the job status and receive a downloaded artifact. This keeps the database safe from massive single-query memory spikes while delivering "the best service" by not forcing clients off the platform for reporting.

---

## 5. The Write Path: Optimized Ingestion

- **Atomic Transactions**: Encapsulate creates and updates within strict database transaction blocks.
- **Chunked Bulk Ingestion**: The `EntriesManager` chunks high-volume multi-row ingestion (e.g., 500 entities per chunk) to prevent OOM crashes and massive lock contention. Transactions are scoped to the chunk.
- **Payload Splitting Engine**: Isolates the core JSON payload for `entry_data` and extracts explicitly indexed fields for the Extension Pages.
- **Idempotent Upserts**: All extension table writes must use `INSERT ... ON DUPLICATE KEY UPDATE` semantics to prevent constraint violations upon replayed events.

---
