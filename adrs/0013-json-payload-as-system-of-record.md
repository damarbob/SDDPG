# 0013 - JSON Payload as Authoritative System of Record

**Status:** Proposed
**Created:** 2026-04-18

## Context

StarDust stores entity data in two places simultaneously: the `fields` JSON column on `entry_data` (the core payload) and typed slot columns across `entry_slots_page_X` tables (the extension layer). This dual-storage design exists because JSON columns provide schema flexibility and complete data preservation, while typed slot columns provide indexed filtering with bounded execution cost.

When data exists in two locations, the system must establish which copy is authoritative. Without a declared authority, divergence between the JSON payload and a slot value — caused by partial writes, failed backfills, or eviction race conditions — creates reconciliation ambiguity: which value is correct? This ambiguity is not theoretical; it arises naturally whenever the Reconciler backfills a queued entry, the Liberator nullifies a tombstoned slot, or a field's `is_filterable` status changes mid-lifecycle.

The architecture's slot eviction model (ADR `0009`) depends on the answer to this question. If extension table slots were the source of truth, nullifying a slot during the Liberator's sweep would constitute permanent data destruction. The Watcher's provisioning, the Reconciler's backfill, and the Liberator's eviction all assume that the JSON payload is complete and extension slots are derived — but this assumption is never stated as a formal invariant.

## Decision

The `entry_data.fields` JSON column is the sole authoritative system of record for all entity field data. Extension table slot columns are treated exclusively as index materializations — projections of the JSON payload optimized for indexed filtering and retrieval. They are not independent data stores.

This invariant — the Strict Projection Rule — has the following operational implications:

- **Writes**: The full `fields` JSON payload is always written to `entry_data` first. Extension table slot writes are a subsequent materialization step. If slot assignment fails (ADR `0007`), the JSON payload is still persisted and the entry is enqueued for backfill.
- **Retrieval**: Demoted or unindexed fields are retrieved via `JSON_EXTRACT(fields, '$.fieldName')` from the core payload. No data is lost when a field's `is_filterable` flag changes or when a slot is evicted.
- **Eviction**: The Liberator can safely nullify slot columns (ADR `0009`) because the nullified data still exists in the JSON payload. Slot eviction never causes permanent data destruction.
- **Reconciliation**: When the JSON payload and a slot value diverge, the JSON payload is canonical. The Reconciler's `INSERT ... ON DUPLICATE KEY UPDATE` always sources values from the JSON payload, not from any cached or intermediate state.

## Consequences

**Positive:**

- Slot eviction is a safe, non-destructive operation. The Liberator's tombstone-sweep-reclaim cycle (ADR `0009`) can nullify slot columns without risk of permanent data loss, because the JSON payload preserves the complete entity.
- Field demotion is seamless. Changing `is_filterable` from `true` to `false` immediately falls back to `JSON_EXTRACT` retrieval without any data migration or backfill.
- Write-availability fallback (ADR `0007`) is data-complete. When all slots are exhausted, the JSON payload is still fully persisted. No field data is lost — only indexed filterability is temporarily degraded.
- The Reconciler's backfill logic is straightforward: read the JSON payload, extract the field value, write it to the assigned slot. The JSON payload is always the canonical source.

**Negative:**

- `JSON_EXTRACT` retrieval is slower than direct column reads from extension tables. Fields that are demoted or not yet backfilled incur higher retrieval latency.
- The JSON payload must always be kept complete and consistent. Any code path that modifies entity data must update the JSON payload — partial updates that modify only extension table slots and skip the JSON column would violate the invariant and create silent data divergence.
- Storage is duplicated: every indexed field value exists in both the JSON payload and an extension table slot. This is a deliberate trade-off for data safety, but it increases storage requirements proportionally to the number of indexed fields.

**Rejected alternatives:**

- Extension tables as source of truth (JSON as backup) — slot eviction or page corruption causes permanent data loss. The Liberator cannot safely nullify slots, and field demotion requires a data migration from the slot back to JSON before the slot can be freed.
- Dual-write with no declared authority — creates reconciliation ambiguity when the JSON payload and a slot value diverge. Every consumer, daemon, and recovery path must independently decide which value to trust, leading to inconsistent behavior across the system.
- JSON-only storage without extension tables — eliminates the dual-storage problem entirely but gives up bounded indexed filtering, which is the core architectural purpose of extension tables (ADR `0001`).
