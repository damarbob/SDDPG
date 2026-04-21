# 0016 - Field Type Change Lifecycle (Type-Driven Slot Reassignment)

**Status:** Accepted
**Created:** 2026-04-20

## Context

StarDust's extension table slots are typed physical columns (`i_str_XX` = VARCHAR, `i_int_XX` = BIGINT, `i_num_XX` = DOUBLE, `i_dt_XX` = DATETIME) whose SQL type is fixed at DDL time. Because `ALTER TABLE` on populated pages is strictly forbidden (§2.1.1 / ADR `0012`), a field's existing slot cannot be altered to accommodate a new type. Nor can the old slot be reused under the new type — the physical column type is incompatible.

The slot assignment lifecycle (§2.1.5) provides the foundation for acquiring a slot of the correct type on any existing page. The tombstone-sweep-reclaim lifecycle (§2.1.3 / ADR `0009`) handles cleanup of the vacated slot. What is unspecified is the composition of these two lifecycles when triggered by a type change, and the two questions that composition raises: who is responsible for backfilling the new slot from the JSON payload, and whether the field should be filterable via the new slot index before backfill is confirmed complete.

## Decision

A field type change triggers the **retype → tombstone → assign → backfill → promote** lifecycle defined in §2.1.6. The following commitments govern its execution:

**Filterability is held for the entire backfill window.** A retype is expressed as coordinated state changes across two `stardust_slot_assignments` rows — the old slot flips `assigned → tombstoned` (and enters the standard Liberator sweep path), and, where capacity allows, a new slot of the target type flips `free → backfilling` in the same transaction (§2.3). There is no intermediate `retyping` slot status; the transitional state of the field is fully expressed by its new slot being in `backfilling` (or by the absence of any live slot, if capacity was unavailable at retype time). While the field's live slot is in `backfilling` — or while the field has no live slot at all — `is_filterable` has no effect on query routing, and all reads fall back to `JSON_EXTRACT`. Filterability is restored only when the Reconciler advances the slot status to `ready`. This prevents filtered queries from silently returning incomplete results during the migration window.

**Backfill is owned by the Reconciler.** The Reconciler already performs chunked JSON-to-slot extraction for queue-drained entries (§2.1.2). Extending it to process type-change backfill records reuses this infrastructure without introducing a new daemon or a new coordination path. Backfill proceeds in configurable chunks with the same throttling and `SKIP LOCKED` mutual exclusion used for sync queue drain.

**Coercion failures store NULL and do not abort the backfill.** Entries whose values cannot be coerced to the new type (e.g., a non-numeric string being retyped to `int`) are stored as `NULL` in the new slot column. The JSON payload remains the authoritative system of record (§2.1.3 / ADR `0013`), so these entries fall back to `JSON_EXTRACT` on retrieval without permanent data loss.

**New page provisioning is not part of the type change operation.** If no free slot of the target type exists on any existing page, the field waits in `JSON_EXTRACT` fallback until the Watcher independently detects low capacity and provisions a new page. The type change does not trigger DDL.

## Consequences

**Positive:**

- No new page is provisioned for a type change unless the target slot type happens to be globally exhausted — existing capacity is used first.
- Holding filterability until backfill is confirmed prevents a class of silent result-set incompleteness bugs that would otherwise be extremely difficult to diagnose.
- Coercion failures degrade gracefully to `JSON_EXTRACT` rather than blocking the type change or causing permanent data loss.
- The Reconciler's existing chunked backfill infrastructure is reused; no new daemon or IPC mechanism is introduced.
- The old slot's cleanup follows the established Liberator path (ADR `0009`) with no special casing.

**Negative:**

- The field is unqueryable by index for the duration of the backfill window. On large tables with millions of rows, this window can span many Reconciler polling cycles.
- Coercion failures are silent at the individual row level. Operators must monitor Reconciler backfill completion reports to identify rows where `NULL` was stored due to incompatible values rather than a genuinely absent field.
- Fields that change type frequently incur repeated tombstone + backfill cycles, each consuming Reconciler capacity and temporarily suspending index-based filterability.

**Rejected alternatives:**

- Immediate slot reuse with type cast at query time — would require the query layer to track mid-flight type migrations and emit type-specific SQL dynamically, leaking migration state into the hot read path and creating fragile, version-dependent behavior.
- Block the API until backfill completes — synchronous backfill on large tables would stall the type change request for seconds to minutes, violating the write-availability principle (§2.1 / ADR `0007`).
- Dedicated backfill daemon — duplicates the Reconciler's existing JSON-to-slot infrastructure without benefit. The Reconciler is already designed for throttled, chunked extension table writes under `SKIP LOCKED` mutual exclusion.
