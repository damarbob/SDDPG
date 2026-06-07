# 0029 - Liberator Sweep Omits the Tenant Predicate

**Status:** Accepted
**Created:** 2026-06-07

## Context

The Liberator's chunked nullification of a tombstoned slot is described across several documents with a `tenant_id` predicate on the `UPDATE`:

- ADR [`0009`](0009-tombstone-based-slot-eviction.md) §Decision step 3: `UPDATE entry_slots_page_X SET i_str_XX = NULL WHERE tenant_id = ? AND id > ? LIMIT 500`.
- Architecture Blueprint §1.2 ("Every query, daemon sweep, and export job carries a `tenant_id` predicate") and §2.1.3 (the same example `UPDATE`).
- `blueprints/liberator_daemon.md` §2 and the §5 flowchart sketch.

The shipped implementation (`StarDust\Liberator\SlotSweeper`) deliberately omits the predicate: it selects the chunk with `WHERE entry_id > ? ORDER BY entry_id LIMIT N` and nullifies with `UPDATE … SET <slot_column> = NULL WHERE entry_id IN (…)`. The same `liberator_daemon.md` blueprint is internally inconsistent on this point — its acceptance criterion AC#3 specifies the sweep as `(page, slot_column) for id > sweep_cursor_id` with **no** tenant predicate, contradicting its own §2 and §5 sketch.

Two facts make the tenant predicate redundant rather than load-bearing:

1. **Single-owner columns.** The `UNIQUE (page_id, slot_column)` constraint on `stardust_slot_assignments` (physically `ux_slot_assignments_page_column`; see [schema_reference §4.4](../schemas/schema_reference.md)) guarantees that any given slot column on a page is mapped to exactly one model, and therefore exactly one tenant. Only that one tenant ever writes non-NULL data into the column. A range `UPDATE … SET <col> = NULL` over `entry_id` touches other tenants' rows, but their cells in that column are already `NULL`, so the write is a no-op for them. Tenant isolation of the *data* is preserved by the schema, not by the predicate.
2. **Tenant is unrecoverable post-tombstone.** A tombstoned slot's `stardust_slot_assignments` row carries `field_id = NULL` ([ADR 0009](0009-tombstone-based-slot-eviction.md) §Tombstone; [schema_reference §4.6](../schemas/schema_reference.md) invariant #1). The field → model → tenant chain is therefore severed before the Liberator ever runs, so the sweep cannot cheaply recover a `tenant_id` to bind even if one were wanted; binding it would require denormalizing `tenant_id` onto the slot-assignment row solely to satisfy a predicate that does no work.

This leaves a documentation conflict — the code follows AC#3 while higher-level documents mandate the predicate — that must be resolved one way. We resolve it in favor of the shipped behavior, which is correct.

## Decision

The Liberator's per-chunk sweep is normatively keyed on **`(page, slot_column)` for `entry_id > sweep_cursor_id`**, with **no `tenant_id` predicate**:

```sql
UPDATE entry_slots_page_X SET i_str_XX = NULL WHERE entry_id > ? ORDER BY entry_id LIMIT 500
```

(The implementation bounds the chunk with a prior `SELECT entry_id … WHERE entry_id > ? ORDER BY entry_id LIMIT N` and nullifies the selected ids; the shape above is the normative intent, not a mandate on statement count.)

`liberator_daemon.md` AC#3 is the normative sweep shape. Its §2 prose and §5 flowchart are corrected to match AC#3.

This decision **refines ADR [`0009`](0009-tombstone-based-slot-eviction.md)**: it overrides the `WHERE tenant_id = ? AND id > ?` predicate in 0009 §Decision step 3 only. Every other commitment of ADR 0009 — the sever → tombstone → sweep → reclaim lifecycle, per-chunk `sweep_cursor_id` checkpointing in the same transaction, the singleton model, the deadlock/gap policy, and `tombstoned_at ASC` ordering — remains in force unchanged. ADR 0009 is **not** superseded; per the document-precedence rule "on conflict between two ADRs, the newer ADR wins," this ADR governs the sweep-predicate question alone, and ADR 0009's body is left intact per the append-only ADR convention.

Because ADRs are the authoritative source of truth and the Architecture Blueprint is a synthesis beneath them ([implementation_phases.md §Document Precedence](../implementation_phases.md)), this ADR also governs over the Architecture Blueprint §1.2/§2.1.3 wording. The Blueprint is updated to carve out the Liberator tombstone sweep as a documented exception that cites this ADR; the §1.2 isolation invariant remains in force for every other query, daemon sweep, and export job.

## Consequences

**Positive:**

- The shipped code, ADR 0009 (as refined here), the Architecture Blueprint, and `liberator_daemon.md` all describe one sweep shape; the precedence inversion and the liberator blueprint's internal §2-vs-AC#3 contradiction are both closed.
- No denormalization of `tenant_id` onto `stardust_slot_assignments` is introduced solely to satisfy a redundant predicate, keeping the registry row minimal and the tombstone state self-consistent (`field_id = NULL` with no shadow tenant column).
- The sweep's correctness now rests explicitly on the `UNIQUE (page_id, slot_column)` invariant, making the dependency auditable: anyone weakening that constraint is on notice that the sweep's tenant isolation depends on it.

**Negative / constraints:**

- The Architecture Blueprint §1.2 "every daemon sweep carries a `tenant_id` predicate" invariant now has one named exception. Readers must consult this ADR to understand why the Liberator is exempt.
- The exemption is contingent on single-owner slot columns. **Any future change that lets a single `(page_id, slot_column)` hold data for more than one tenant** (e.g., a cross-tenant page-packing scheme) would reintroduce data-bleed risk during sweep and MUST revisit this ADR and restore a tenant-scoping mechanism.

**References:** ADR [`0009`](0009-tombstone-based-slot-eviction.md) (refined here), ADR [`0017`](0017-schema-registry-as-coordination-contract.md) (registry as coordination contract; the slot-assignment uniqueness and `field_id` nullability invariants this decision relies on).
