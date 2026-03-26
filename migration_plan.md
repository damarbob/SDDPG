# Vertical Partitioning Migration Plan

This document outlines the strategic plan for migrating StarDust from the current **Virtual Column Method** to the **Vertical Schema Partitioning (Extension Tables)** architecture.

## 1. Objectives

- **Architectural Shift**: Move from single-table virtual columns to a 1:1 extension table strategy (`entry_data` + `entry_slots_page_X`), optimized strictly as a high-throughput ingestion and discrete retrieval engine.
- **Scale Without Spillage**: Ensure MySQL query execution is tightly bounded via a configurable circuit breaker, preventing disk spillage of temporary tables by seamlessly offloading arbitrary reporting to an asynchronous search layer.
- **Stable Consumer Contract**: Provide explicit, deterministic API endpoints that gracefully handle scale without forcing clients to write brittle fallback logic.
- **Decoupled Migration**: Structure the migration with an asynchronous Dual-Write strategy using causally ordered event streams, completely decoupling the legacy system's uptime from the new schema's experimental stability.
- **Operational Resilience**: Provide quantifiable cutover criteria, idempotent data pipelines, guaranteed data integrity under degraded states, and instant rollback capability.

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
  - **⚠️ Selective Indexing**: Indexes are governed by the Index Provisioning Policy (§2.2). Arbitrary reporting is offloaded to the search infrastructure.

> [!NOTE]
> **Slot Layout Change from Original Analysis:** v6 intentionally revised the per-page slot distribution from the original analysis (`i_str_30`, `i_num_20`, `i_dt_10`) to include a dedicated integer type: `i_str_25`, `i_int_15`, `i_num_10`, `i_dt_10`. Rationale: foreign-key lookups and enum-coded fields benefit from native `BIGINT` indexing over `DOUBLE` casting.

### 2.1 Automated Page Provisioning & Exhaustion Fallback

A background daemon monitors global slot consumption and provisions new extension pages when capacity drops below 20%.

- **Safeguards**: Employs Empty-Table-Only DDL, Advisory Locking (`GET_LOCK('stardust_page_provision', 10)`), and Atomic Registry Updates to prevent metadata lock contention. `ALTER TABLE` on populated pages via daemon is strictly forbidden.
- **⚠️ Exhaustion Fallback (Resilience)**: If the provisioning daemon fails and slot capacity reaches 100%, high-throughput ingestion **must not block or drop writes**.
  - The payload splitting engine will gracefully degrade.
  - It writes the full JSON payload into the `entry_data` table and skips writing to the `entry_slots_page_X` table. The failed operation ID is written into the `stardust_sync_queue` table.
  - **Persistent Queue Daemon**: A background PHP daemon continuously polls the `stardust_sync_queue` utilizing `SELECT ... FOR UPDATE SKIP LOCKED` to concurrently claim tasks without metadata lock contention on the main ingestion thread.
  - **Reconciler Backpressure**: Once a new page is provisioned, the daemon aligns the pending entries into the new extension table and deletes the queue rows. It processes in configurable chunks (default: `500`) with a configurable delay between chunks to prevent write spikes against the extension table during recovery. The daemon reports throughput metrics to stdout (rows processed / elapsed time).

### 2.2 Index Provisioning Policy

Indexing decisions are **schema-driven**, governed by the `is_filterable` metadata flag set at model-field registration time.

- If `is_filterable = true` → a composite B-tree index `(tenant_id, slot_column)` is included in the page DDL at provisioning time.
- If `is_filterable = false` → the slot is used for discrete retrieval only. **Filters against non-filterable fields are strictly forbidden.** To maintain low latency, MySQL will never evaluate `JSON_EXTRACT` inside a `WHERE` clause. If a consumer attempts to filter on an unindexed field, the API will strictly reject the request with an **HTTP 400 Bad Request**. No fallback or suggestions for in-memory bulk filtering will be provided, forcing the consumer to request proper schema provisioning if the filter is a genuine business requirement.
- This ensures index provisioning is a deterministic, auditable decision tied to the schema registry.

---

## 3. The API Contract & Consumer Abstraction

The API must never leak the physical database design or silently shift consistency models without explicit headers.

- **API Purity & Predictability**:
  - Consumers use `/api/entries` for standard synchronous data retrieval.
  - To minimize the maintenance layer, relying on external 3rd-party services (like OpenSearch) is strongly discouraged. As a standalone engine, StarDust does not perform transparent proxying or dynamic shifts to eventual consistency.
  - To support scale without breaking the consumer contract, `/api/entries` mandates **Cursor-Based Pagination**. The system never evaluates the total matched set of a query; it only evaluates the requested page (e.g., `LIMIT 50`). This ensures valid business queries are never arbitrarily rejected with 422 errors simply because the result set grew over time.
  - If a synchronous query scans an excessively large amount of rows without a proper index limit, it will trip the **Scanned Row Circuit Breaker** (see Section 4).
- **Seamless Field Selection**: Consumers request fields logically using standard query parameters (e.g., `?select=firstName,jobId`). The system dynamically determines if a field requires an `INNER JOIN` against an extension table or a `JSON_EXTRACT(fields, '$.propertyName')` from the core payload on retrieval (select only, never for filtering), automatically merging the final representation.

---

## 4. The Read Path: Bounded Query Execution

The CodeIgniter 4 query builder ensures InnoDB is protected from runaway materialize operations without punishing consumers for large datasets.

- **Tenant Isolation**: Secure all `INNER JOIN` conditions by enforcing `tenant_id` matches across all pages.
- **Deterministic Late Row Lookups (Two-Query Approach)**: Cross-page queries and complex filters must never be executed as a single, potentially unbound `INNER JOIN`. To guarantee bounded execution and prevent disk spillage, the executor enforces a strict two-step process:
  1.  **Query 1 (Paginated Probe)**: Execute the filter condition as a standalone query selecting _only_ `id` using covering indexes, bounding the query using standard cursor logic: `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. This determines if a next page exists while maintaining a constant, tiny memory footprint regardless of total matched rows.
  2.  **Query 2 (Bounded Fetch)**: Use the resulting array of IDs (up to `page_size`) to fetch the full row payloads via a safe `WHERE id IN (...)` clause with any necessary joins.

### 4.1 Scanned Row Circuit Breaker

The read path utilizes a circuit breaker, changing its focus from "punishing large match counts" to "preventing unindexed table scans."

- **Threshold Trigger**: Rather than tripping on the number of returned matches, the system evaluates internal limits or tracked metrics to ensure the number of _scanned rows_ does not exceed a hard threshold.
- **Hard Failure Fallback**: If a query is poorly formed (e.g., forcing a massive scan despite indices), the framework aborts the request and returns an **HTTP 400 Bad Request** outlining that the query is non-performant. It does _not_ throw 422 errors for queries that successfully use an index but happen to match many rows.

### 4.2 Asynchronous Exports (Massive Data Retrieval)

To maintain the strict synchronous page limits on `/api/entries` without abandoning consumers who genuinely need massive datasets, StarDust introduces a dedicated export pattern.

- **The `/api/exports` Endpoint**: A consumer posts a large query filter to `/api/exports`. The API accepts the payload, validates the query, and responds immediately with a `202 Accepted` and an export Job ID.
- **Background Materialization**: A PHP daemon (akin to the sync queue) picks up the export job. It seamlessly pages through the database using the same Cursor-Based Pagination described above (Query 1 and Query 2 in a loop) and writes the output locally to a streaming CSV or JSON file on disk.
- **Result Delivery**: Once complete, the consumer can poll the job status and receive a downloaded artifact. This keeps the database safe from massive single-query memory spikes while delivering "the best service" by not forcing clients off the platform for reporting.

---

## 5. The Write Path: Optimized Ingestion

- **Atomic Transactions**: Encapsulate creates and updates within strict database transaction blocks.
- **Chunked Bulk Ingestion**: The `EntriesManager` chunks high-volume multi-row ingestion (e.g., 500 entities per chunk) to prevent OOM crashes and massive lock contention. Transactions are scoped to the chunk.
- **Payload Splitting Engine**: Isolates the core JSON payload for `entry_data` and extracts explicitly indexed fields for the Extension Pages.
- **Idempotent Upserts**: All extension table writes must use `INSERT ... ON DUPLICATE KEY UPDATE` semantics to prevent constraint violations upon replayed events.

---
