# 0009 - Tombstone-Based Slot Eviction over Immediate Reuse

**Status:** Proposed
**Created:** 2026-04-16

## Context

When a field is deleted or demoted from `is_filterable = true` to `false`, its physical slot in an extension table (e.g., `i_str_03` on `entry_slots_page_2`) is no longer needed for indexed filtering. The slot column still contains data from the previous occupant across potentially millions of rows. If the system immediately maps this slot to a new field, the new field inherits stale values from the old one — a condition called "data bleeding" — causing silent data corruption that is extremely difficult to detect after the fact because the values are type-compatible and structurally valid.

The extension-table layer has finite capacity (ADR `0007`). Tombstoned slots temporarily reduce available capacity, which can accelerate slot exhaustion and trigger the write-availability fallback. The system must balance the urgency of reclaiming capacity against the data-correctness risk of premature reuse.

The Strict Projection Rule (§2.1.3) provides the safety foundation: extension table slots are materializations of the `fields` JSON payload, not independent stores. A demoted field's data is never lost — it remains accessible via `JSON_EXTRACT` from the core payload. This means the eviction process can safely nullify slot values without risking permanent data destruction.

## Decision

Slot eviction follows a strict three-phase lifecycle: **sever → tombstone → sweep → reclaim**.

1. **Sever.** When a field is deleted or its `is_filterable` flag is demoted, its mapping in the schema registry is immediately severed. The API instantly falls back to `JSON_EXTRACT(fields, '$.fieldName')` for subsequent retrieval queries involving that field. No queries will reference the physical slot column from this point forward.

2. **Tombstone.** The severed slot is marked `tombstoned` in the registry. A tombstoned slot cannot be mapped to a new field for that model. It is effectively quarantined — visible to the Liberator daemon but invisible to the provisioning path.

3. **Sweep.** The Liberator daemon lazily processes tombstoned slots via chunked DML nullification (`UPDATE entry_slots_page_X SET i_str_XX = NULL WHERE tenant_id = ? AND id > ? LIMIT 500`). This runs without table-level locks and is throttled to avoid write spikes.

4. **Reclaim.** Once the Liberator confirms a slot is 100% nullified for a given tenant/model partition, it updates the registry to mark the slot `free`. Only then is the slot safely available for mapping to a new field.

### Operational Parameters

The Liberator's runtime profile is fixed by this ADR to give the Watcher a stable capacity-accounting model and to make sweep behavior auditable.

- **Singleton.** The Liberator runs as a single instance per deployment, enforced by a PID file or OS-level process lock identical to the Watcher's mechanism (see ADR `0008`). Multiple Liberator workers are explicitly rejected: concurrent `UPDATE ... SET i_str_XX = NULL` statements against the same `(page, slot)` produce no correctness problem (NULL is idempotent), but they multiply InnoDB row-lock contention and obscure sweep progress in the per-slot `sweep_cursor_id`. The sweep workload is dominated by IO and short row locks, not CPU; horizontal scaling delivers no throughput benefit and degrades observability.
- **Per-batch checkpoint.** The Liberator commits the updated `sweep_cursor_id` (next-batch high-water mark on `entry_id`) to `stardust_slot_assignments` in the **same transaction** as the chunked `UPDATE`. There is no separate "save cursor" step. On crash, the Liberator resumes from the last committed `sweep_cursor_id` for that slot and re-issues the next chunk; already-nullified rows in the resumed chunk are no-ops because `NULL` is idempotent.
- **Deadlock policy.** Concurrent reads against the swept page take only shared row-level locks (the bounded fetch in ADR `0005` reads via `WHERE id IN (...)`), so deadlocks against the Liberator's `UPDATE ... LIMIT 500` are uncommon but possible under high read load on the same `tenant_id` partition. On `SQLSTATE 40001` (deadlock detected by InnoDB), the Liberator rolls back the chunk, sleeps for the inter-chunk delay, and retries from the same `sweep_cursor_id` — it does **not** advance the cursor on failure. After three consecutive deadlocks against the same chunk, the Liberator logs the slot identity and skips ahead by `LIMIT` rows, leaving a `sweep_gap` flag on the registry row for operator inspection. This bounded retry prevents a hot read partition from indefinitely blocking sweep progress while still surfacing pathological contention.
- **Sweep priority.** The Liberator processes tombstoned slots in `tombstoned_at ASC` order — oldest first — so capacity is reclaimed in roughly the order it was vacated. Tie-broken by `(page_id, slot_column)` for deterministic ordering across restarts.

## Consequences

**Positive:**

- Data bleeding is structurally impossible. A new field can only be mapped to a slot that has been verified empty, eliminating an entire class of silent data corruption.
- The eviction process is non-blocking. Severing the registry mapping is instantaneous; the API falls back to `JSON_EXTRACT` without waiting for the sweep to complete. Retrieval is never interrupted.
- No data is permanently destroyed. The JSON payload remains the system of record (ADR `0013`), so nullifying a slot column is a safe, reversible operation.
- The Liberator operates as an isolated failure domain. If it crashes, tombstoned slots simply remain quarantined — capacity is reduced but correctness is preserved.

**Negative:**

- Tombstoned slots temporarily squat on finite capacity. Under high field churn (frequent promotions and demotions), the number of simultaneously tombstoned slots can grow, reducing available capacity and increasing pressure toward slot exhaustion.
- The Liberator's sweep latency is non-deterministic. Large tables with millions of rows require many chunked passes, during which the slot remains unavailable.
- Operators must monitor tombstone depth and Liberator throughput as leading indicators of capacity pressure. A stalled Liberator silently degrades available capacity without triggering any immediate error.
- The singleton model means total sweep throughput is bounded by one process. Operators with extreme tombstone churn must accept longer reclaim latency rather than horizontally scaling the daemon.

**Rejected alternatives:**

- Immediate slot reuse after demotion — causes data bleeding if any rows still contain the old field's values. The new field would silently inherit stale data that is type-compatible and structurally indistinguishable from legitimate values.
- `DELETE` rows containing stale data and re-insert — destructive, risks permanent data loss if the JSON payload is incomplete, and causes massive write amplification on large tables.
- Out-of-band cleanup before reuse (block provisioning until sweep completes) — adds latency to the provisioning path and couples the Watcher's DDL cycle to the Liberator's sweep progress, violating failure isolation.
- Multi-worker Liberator coordinated by `SKIP LOCKED` on `stardust_slot_assignments` — rejected because the sweep workload is IO-bound (bulk `UPDATE` on a single page) rather than CPU-bound, so additional workers do not increase throughput. They do, however, multiply InnoDB row-lock contention against concurrent reads on the swept page and fragment the per-slot `sweep_cursor_id` semantics. The marginal throughput gain does not justify the loss of a clean single-cursor sweep model.
- Periodic checkpoint flush (e.g., every N chunks rather than per-chunk) — rejected because the per-chunk checkpoint is already committed alongside the chunk's `UPDATE` in the same transaction, making the checkpoint write effectively free. A separate flush cadence would create a window where committed nullifications are not reflected in the registry, allowing the Watcher to undercount free capacity after a Liberator crash.
