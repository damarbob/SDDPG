# Vertical Partitioning Migration Plan

This document outlines the strategic plan for migrating StarDust from the current **Virtual Column Method** to the **Vertical Schema Partitioning (Extension Tables)** architecture.

## 1. Objectives

*   **Architectural Shift**: Move from single-table virtual columns to a 1:1 extension table strategy (`entry_data` + `entry_slots_page_X`), optimized strictly as a high-throughput ingestion and discrete retrieval engine.
*   **Scale Without Spillage**: Ensure MySQL query execution is tightly bounded via a configurable circuit breaker, preventing disk spillage of temporary tables by seamlessly offloading arbitrary reporting to an asynchronous search layer.
*   **Stable Consumer Contract**: Provide explicit, deterministic API endpoints that gracefully handle scale without forcing clients to write brittle fallback logic.
*   **Decoupled Migration**: Structure the migration with an asynchronous Dual-Write strategy using causally ordered event streams, completely decoupling the legacy system's uptime from the new schema's experimental stability.
*   **Operational Resilience**: Provide quantifiable cutover criteria, idempotent data pipelines, guaranteed data integrity under degraded states, and instant rollback capability.

---

## 2. Technical Specifications & Storage Strategy

MySQL is treated strictly as a transactional store, not a catch-all search engine.

*   **`entry_data` (Core Payload Table)**
    *   `id` (BIGINT, Primary Key)
    *   `tenant_id` (BIGINT, Index for tenant isolation)
    *   `model_id` (INT)
    *   `created_at`, `updated_at`, `deleted_at` (DATETIME, System timestamps)
    *   `fields` (JSON, Complete unindexed payload)
    *   `is_desynced` (TINYINT(1), Default 0) - Indicates if the extension table write was skipped due to full capacity.
    *   *Indexes*: `(tenant_id, model_id)`, `(tenant_id, deleted_at, created_at)`, `(is_desynced)`
*   **`entry_slots_page_X` (1:1 Extension Tables)**
    *   `entry_id` (BIGINT, Primary Key / Foreign Key `ON DELETE CASCADE`)
    *   `tenant_id` (BIGINT)
    *   `i_str_01`...`i_str_25` (VARCHAR), `i_int_01`...`i_int_15` (BIGINT), `i_num_01`...`i_num_10` (DOUBLE), `i_dt_01`...`i_dt_10` (DATETIME)
    *   **⚠️ Selective Indexing**: Indexes are governed by the Index Provisioning Policy (§2.2). Arbitrary reporting is offloaded to the search infrastructure.

> [!NOTE]
> **Slot Layout Change from Original Analysis:** v6 intentionally revised the per-page slot distribution from the original analysis (`i_str_30`, `i_num_20`, `i_dt_10`) to include a dedicated integer type: `i_str_25`, `i_int_15`, `i_num_10`, `i_dt_10`. Rationale: foreign-key lookups and enum-coded fields benefit from native `BIGINT` indexing over `DOUBLE` casting.

### 2.1 Automated Page Provisioning & Exhaustion Fallback

A background daemon monitors global slot consumption and provisions new extension pages when capacity drops below 20%.

*   **Safeguards**: Employs Empty-Table-Only DDL, Advisory Locking (`GET_LOCK('stardust_page_provision', 10)`), and Atomic Registry Updates to prevent metadata lock contention. `ALTER TABLE` on populated pages via daemon is strictly forbidden.
*   **⚠️ Exhaustion Fallback (Resilience)**: If the provisioning daemon fails and slot capacity reaches 100%, high-throughput ingestion **must not block or drop writes**.
    *   The payload splitting engine will gracefully degrade.
    *   It writes the full JSON payload into the `entry_data` table, skips writing to the `entry_slots_page_X` table, and sets `is_desynced = 1`.
    *   A background reconciler cron job continuously monitors for `is_desynced = 1`. Once a new page is provisioned by operators, the reconciler processes these flagged rows, aligns them into the new table, and clears the flag.
    *   **Reconciler Backpressure**: The reconciler **must** be cursor-based: `WHERE is_desynced = 1 ORDER BY id ASC LIMIT {chunk_size}` (configurable, default: `500`). It processes in configurable chunks with a configurable delay between chunks to prevent write spikes against the extension table during recovery. It reports throughput metrics to stdout (rows processed / elapsed time). This mirrors the backfill pump architecture in §6.3.

### 2.2 Index Provisioning Policy

Indexing decisions are **schema-driven**, governed by the `is_filterable` metadata flag set at model-field registration time.

*   If `is_filterable = true` → a composite B-tree index `(tenant_id, slot_column)` is included in the page DDL at provisioning time.
*   If `is_filterable = false` → the slot is used for discrete retrieval only. Queries filtering against non-filterable fields are routed to `JSON_EXTRACT(fields, '$.propertyName')` for small result sets, or to the Search Layer for arbitrary reporting.
*   This ensures index provisioning is a deterministic, auditable decision tied to the schema registry, not operator judgment.

---

## 3. The API Contract & Consumer Abstraction

The API must never leak the physical database design or silently shift consistency models without explicit headers.

*   **Unified Routing & Gateway Proxying**:
    *   Consumers exclusively use `/api/entries`.
    *   If the requested filters are simple (e.g., maximum of 2 `entry_slots_page_X` joins) and the match count is low, the Database Engine synchronously returns the payload (Strict Consistency).
    *   If the query is highly complex or hits the **Circuit Breaker** (see Section 4), the API Gateway **automatically proxies** the exact request parameters to the backend Search Service (e.g., OpenSearch / `/api/search/entries`).
    *   When proxying, the Gateway appends an `X-Consistency-Model: Eventual` header to the response, informing the consumer of the slight replication delay without breaking frontend integration.
    *   **Proxy Timeout & Error Normalization**: The gateway enforces a hard timeout of `stardust.gateway.proxy_timeout_ms` (default: `5000`). Any non-2xx response **or** timeout from the Search Layer is treated identically to a hard failure — the gateway returns the 422 `ERR_TOO_MANY_MATCHES` response defined in §4.1. The gateway must **never** forward a raw upstream error to the consumer. Upstream failures are logged internally for operators.
*   **Seamless Field Selection**: Consumers request fields logically using standard query parameters (e.g., `?select=firstName,jobId`). The system dynamically determines if a field requires an `INNER JOIN` against an extension table or a `JSON_EXTRACT(fields, '$.propertyName')` from the core payload, automatically merging the final representation.

---

## 4. The Read Path: Bounded Query Execution

The CodeIgniter 4 query builder ensures InnoDB is protected from runaway materialize operations.

*   **Tenant Isolation**: Secure all `INNER JOIN` conditions by enforcing `tenant_id` matches across all pages.
*   **Deterministic Late Row Lookups (Two-Query Approach)**: Cross-page queries and complex filters must never be executed as a single, potentially unbound `INNER JOIN`. To guarantee bounded execution and prevent disk spillage, the executor enforces a strict two-step process:
    1.  **Query 1 (Lightweight Probe)**: Execute the filter condition as a standalone query selecting *only* `id` using covering indexes, appended with `LIMIT {max_intermediate_rows} + 1`. This is cheap and immune to optimizer inlining issues.
    2.  **Query 2 (Bounded Fetch)**: If Query 1 yields a count within the allowed threshold, use the resulting array of IDs to fetch the full row payloads via a safe `WHERE id IN (...)` clause with any necessary joins.

### 4.1 Configurable Circuit Breaker

Before performing the wide fetch (Query 2), the execution engine evaluates the result count of the lightweight probe (Query 1).

*   **Threshold Trigger**: If Query 1 yields more rows than `stardust.query.max_intermediate_rows` (default: `50000`), the query is aborted on the MySQL side, and a Circuit Breaker exception is thrown.
*   **Gateway Handling**: As defined in Section 3, the API Controller catches this exception and proxies the request to the Search Layer.
*   **Hard Failure Fallback**: If the proxying fails (e.g., the Search infrastructure is completely offline, returns a non-2xx, or exceeds the proxy timeout), the API must **abort** and return an **HTTP 422 Unprocessable Entity** (Payload is syntactically valid but exceeds operational limits) with a specific application code: `{"code": "ERR_TOO_MANY_MATCHES", "message": "Query exceeded maximum internal bounds. Please narrow your filters."}`.

---

## 5. The Write Path: Optimized Ingestion

*   **Atomic Transactions**: Encapsulate creates and updates within strict database transaction blocks.
*   **Chunked Bulk Ingestion**: The `EntriesManager` chunks high-volume multi-row ingestion (e.g., 500 entities per chunk) to prevent OOM crashes and massive lock contention. Transactions are scoped to the chunk.
*   **Payload Splitting Engine**: Isolates the core JSON payload for `entry_data` and extracts explicitly indexed fields for the Extension Pages.
*   **Idempotent Upserts**: All extension table writes must use `INSERT ... ON DUPLICATE KEY UPDATE` semantics to prevent constraint violations upon replayed events.

---


