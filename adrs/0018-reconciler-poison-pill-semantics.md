# 0018 - Reconciler Poison-Pill Semantics: Per-Row DLQ With Chunk-Level Commit

**Status:** Proposed
**Created:** 2026-04-22

## Context

The Reconciler claims work in chunks via `SELECT ... FOR UPDATE SKIP LOCKED` and processes each chunk inside a single database transaction (§2.1.2). Each chunk reads the authoritative `entry_data.fields` payload at upsert time, extracts the indexed-field values, and writes them to the appropriate `entry_slots_page_X` row. On chunk commit, the corresponding rows are removed from `stardust_sync_queue`.

This works cleanly when every row in a chunk is processable. The unresolved question is what happens when a single row in a chunk cannot be processed at all — for example, the JSON payload at `entry_data.fields` is malformed, or the referenced `entry_data` row has been hard-deleted between enqueue and drain, or a schema invariant is violated in a way that cannot be salvaged by ADR `0016`'s coercion fallback. ADR `0016` already covers the benign case (a value that cannot be coerced to the new declared type stores `NULL` in the new slot and falls back to `JSON_EXTRACT`); this ADR addresses the case where there is no graceful fallback.

The naive options collapse into two failure modes that the architecture cannot tolerate:

- **Chunk-level rollback.** A single bad row aborts the entire chunk transaction. `SKIP LOCKED` releases its hold; the next polling cycle claims the same chunk; the same bad row triggers the same rollback. The poison row indefinitely blocks the subset of `stardust_sync_queue` that overlaps it, which under exhaustion-fallback bursts can mean tens of thousands of valid entries waiting behind a single corrupt payload.
- **Row-level skip with queue delete.** The Reconciler logs the bad row, skips it, commits the rest of the chunk, and deletes the entire chunk from `stardust_sync_queue`. The bad row's data lives only in `entry_data.fields` (still authoritative per ADR `0013`), but its indexed materialization will never appear in any extension page, and the system has no record that materialization is owed. The desync becomes silent and undetectable without a full re-scan.

Both failure modes are unacceptable. The Reconciler also drains bulk-import jobs (ADR `0011`), so a poison-pill mechanism must cover both workloads.

## Decision

Per-row failures are quarantined into a dedicated **Reconciler Dead Letter Queue** table. The chunk's transaction commits its remaining rows and atomically moves the failed row from the source queue into the DLQ — never rolls back, never silently deletes.

### `stardust_reconciler_dlq` Table

Added to the Schema Reference §3 alongside `stardust_sync_queue`. One row per quarantined entry. The table is small (DLQ depth is alerted; see "Operational thresholds" below) and rarely written to.

| Column                  | Type                                                                       | Description                                                                                              |
| :---------------------- | :------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| `id`                    | `BIGINT`                                                                   | Primary Key.                                                                                             |
| `source`                | `ENUM('sync_queue','bulk_import')`                                         | Workload that produced the failure. Lets operators triage the two Reconciler workloads independently.    |
| `entry_id`              | `BIGINT` **NULL**                                                          | Authoritative entry pointer. `NULL` only when `reason = 'missing_entry_data'`.                           |
| `tenant_id`             | `BIGINT`                                                                   | Denormalized from `entry_data` at quarantine time. Survives `entry_data` row deletion.                   |
| `model_id`              | `INT`                                                                      | Same.                                                                                                    |
| `reason`                | `ENUM('malformed_json','missing_entry_data','schema_incompatibility','other')` | Closed cause taxonomy. Free-form detail goes in `error_message`.                                       |
| `error_message`         | `TEXT`                                                                     | Sanitized exception class + message. MUST NOT include unbounded payload contents (PII risk).             |
| `failed_at`             | `DATETIME`                                                                 | Quarantine timestamp. Drives the "oldest DLQ entry" operator alert.                                      |
| `retry_count`           | `INT` (default `0`)                                                        | Bumped by the replay CLI on each operator-initiated re-enqueue. Pure audit field; not used for routing.  |
| `chunk_correlation_id`  | `VARCHAR(36)`                                                              | The Reconciler's per-chunk UUID (per ADR `0020`). Lets operators trace from a structured-log event to the DLQ row. |

**Indexes:**

- `INDEX (source, failed_at)` — supports the "oldest unresolved entry per workload" alert query.
- `INDEX (entry_id)` — supports operator triage and replay.

There is no foreign key to `entry_data` — by construction the DLQ MUST survive `entry_data` row deletion (the `missing_entry_data` reason exists precisely for this).

### Quarantine Semantics

Inside the chunk transaction, when the Reconciler encounters a row it cannot process:

1. The Reconciler emits a `chunk_partial` event (per ADR `0017`'s read of ADR `0020`'s event vocabulary), then a `dlq_inserted` event carrying `{source, entry_id, reason, chunk_correlation_id}`.
2. The Reconciler `INSERT`s a row into `stardust_reconciler_dlq` with the failure metadata.
3. The Reconciler `DELETE`s the corresponding row from `stardust_sync_queue` (or marks the bulk-import chunk row `failed`, depending on workload).
4. Processing of the remaining rows in the chunk continues.

The chunk commits as a single transaction. Either every survivor in the chunk is durable AND every poison row is in the DLQ AND every poison row's source-queue entry is removed — or none of the above are durable and the chunk re-claims on the next cycle. There is no observable window in which a poison row sits in both the source queue and the DLQ, or in neither.

### Replay

Replay is **operator-initiated**, never automatic. A CLI command (`php spark stardust:reconciler:dlq:replay --id=N` or `--reason=...` for batch) re-inserts the DLQ row's `entry_id` into `stardust_sync_queue`, increments `retry_count`, and deletes the DLQ row. Both writes commit in one transaction. The Reconciler then drains the re-enqueued row through its standard polling cycle.

Operators are expected to investigate the underlying cause before replaying — replaying a row whose `entry_data.fields` is still malformed will simply re-quarantine it. Pure-loop avoidance is the operator's responsibility, not the Reconciler's; an automatic replay path would re-introduce the chunk-rollback poison-pill behavior this ADR rejects.

### Operational Thresholds

The DLQ is silent unless monitored. Per the structured-log mandate (ADR `0020`), operators MUST configure alerts on:

- **Depth alert:** `COUNT(*) FROM stardust_reconciler_dlq` exceeds `100`. Indicates a systemic ingestion issue producing many poison pills.
- **Age alert:** `MAX(NOW() - failed_at)` exceeds `12 hours` for any unresolved row. Indicates an operator has not triaged stale failures.

Both thresholds are deployment-tunable but the existence of monitoring on both is non-negotiable — the entire decision rests on operators noticing DLQ accumulation, since the Reconciler will not block on it.

## Consequences

**Positive:**

- The Reconciler never deadlocks on a poison row. A single corrupt payload cannot block the queue.
- The Reconciler never silently loses materialization owed to a row. Every poison row has a durable record in `stardust_reconciler_dlq` with sufficient context to triage and replay.
- The chunk-transaction model is preserved unchanged. Survivors commit atomically with the quarantine of their poison neighbors.
- The DLQ is a queryable operator surface. `SELECT reason, COUNT(*) FROM stardust_reconciler_dlq GROUP BY reason` is a complete failure-mode summary; no log archaeology required.
- Both Reconciler workloads (sync-queue drain, bulk-import drain per ADR `0011`) share one DLQ surface, distinguished by the `source` column. Operators learn one tool.
- The replay path is explicit and audited. `retry_count` makes "we replayed this five times and it still fails" visible at a glance.

**Negative:**

- The DLQ is yet another table operators must back up, monitor, and prune. The volume is bounded by the alert thresholds (a healthy system has near-zero DLQ depth), so cost is modest, but it is non-zero.
- Replay is manual. A poison row caused by a transient infrastructure glitch (e.g., a brief disk error during JSON read) requires operator intervention rather than self-healing. The trade-off is intentional: automatic replay re-introduces the deadlock failure mode for permanent errors.
- The closed `reason` ENUM constrains the failure taxonomy. New failure classes require a schema migration. Free-form `error_message` partially compensates, but dashboards and runbooks built against the ENUM must be updated when the taxonomy expands.
- The DLQ holds quarantined rows indefinitely until an operator acts. There is no automatic TTL — purging unresolved failures by time would re-create the silent-data-loss failure mode this ADR rejects.

**Rejected alternatives:**

- **Chunk-level rollback on first failure** — re-introduces the poison-pill deadlock the ADR exists to prevent. Mathematically equivalent to "the queue stops on the first bad row."
- **Row-level skip with queue delete and a log line** — re-introduces silent loss of materialization commitment. The `entry_data.fields` payload is still authoritative (ADR `0013`), but the system has no record that materialization is owed, so the desync is undetectable without a full re-scan.
- **In-memory DLQ (process-local)** — loses everything on Reconciler restart. The Reconciler's restart is the most common operator action; an in-memory DLQ would routinely lose poison rows during normal operations.
- **Automatic exponential-backoff replay** — fails closed for transient errors, fails open for permanent errors (replays a malformed JSON forever, accumulating retry events but never resolving). Operator-initiated replay separates the decision ("is this row actually fixable?") from the mechanism ("re-enqueue and try again").
- **Per-workload DLQ tables** (separate `stardust_sync_queue_dlq` and `stardust_bulk_import_dlq`) — duplicates the schema and the replay tool without benefit. The `source` discriminator on a single table provides identical operational ergonomics with half the surface area.
- **Stash the failed payload alongside the DLQ row** — tempting for offline triage but turns the DLQ into a potential PII pool. The `entry_data.fields` payload remains the authoritative copy (ADR `0013`); the DLQ links to it by `entry_id` and stores only sanitized error metadata.

## Related

- ADR `0011` — Chunked Bulk Ingestion (the second Reconciler workload covered by this DLQ)
- ADR `0013` — JSON Payload as System of Record (why partial-skip is unsafe — the materialization commitment is real)
- ADR `0015` — Database as Sole Daemon Coordination Point (DLQ is a registry-adjacent table; no message bus)
- ADR `0016` — Field Type Change Lifecycle (coercion failures are NOT poison pills — they store NULL and fall back to JSON_EXTRACT)
- ADR `0017` — Schema Registry as Coordination Contract (uniform atomicity model for chunk + DLQ commit)
