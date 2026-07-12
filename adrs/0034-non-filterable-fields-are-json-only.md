# 0034 - Non-Filterable Fields Are JSON-Only (No Slot Residency)

**Status:** Proposed
**Created:** 2026-07-12

## Context

ADR `0003` decided that index provisioning is schema-driven via the `is_filterable` flag, but its Decision section left a side door open: *"When a field is not marked filterable, the slot may still exist for typed retrieval, but it is not considered valid for filtering."* Nothing in the implementation ever closed that door, and nothing ever walked through it either — the result is an inconsistency discovered during a 2026-07-12 design review:

- **No reservation path checks `is_filterable`.** `SlotReserver::reserve()` selects candidates purely by `slot_type` family; `RetypeInitiator` reserves a replacement slot for *every* retype, explicitly passing `requireIndexed: false` when the field is non-filterable; the write path (`LiveSlotMap` + `PayloadSplitter`) materializes into any live slot regardless of filterability. A non-filterable field *can* occupy a slot, and several paths actively give it one.
- **The read path never consults a non-filterable slot.** `FieldDescriptor::isIndexedNow()` requires `isFilterable === true`, and `BoundedFetch` only LEFT-JOINs pages for `isIndexedNow()` fields. A non-filterable field is always served from the `entry_data.fields` JSON payload — even when it holds a live, fully-written slot. The "typed retrieval" allowance was never implemented, and implementing it would be a negative-sum change: the JSON payload is already fetched for fallback assembly, so slot-serving these fields would *add* per-page joins to avoid decoding a column already in memory.
- **The exhaustion fallback spins forever on them.** ADR `0007`'s enqueue fires for any *registered* field lacking a live slot, filterable or not. Writing an entry that touches a slot-less non-filterable field therefore enqueues `stardust_sync_queue` rows whose backfill can never serve a purpose — and because the queue row stays claimable until a slot exists, the Reconciler emits `capacity_wait` indefinitely for a slot nobody should reserve.

Meanwhile ADR `0013` (the Strict Projection Rule) already frames slots as *"exclusively index materializations"* and prescribes `JSON_EXTRACT` retrieval for unindexed fields, and ADR `0007`'s stated purpose for the backfill queue is restoring *"indexed queryability"*. The permissive reading of ADR `0003` and the strict reading of ADR `0013` cannot both be the contract. Slots are also the architecture's scarcest inventory: pages hold a fixed 60-slot layout, page count grows monotonically (ADR `0012`), and spread economics (ADR `0031`/`0032`/`0033`) are computed over *filterable* slots — inventory spent on fields the read path never consults accelerates exhaustion for the fields that matter.

## Decision

A non-filterable field is **JSON-only**: it resides exclusively in the `entry_data.fields` payload and is **never assigned an extension-table slot**. Extension slots are reserved for filterable fields, hardening ADR `0013`'s "exclusively index materializations" phrasing into an enforced invariant. ADR `0003`'s "typed retrieval" allowance is withdrawn (annotated in place there, pointing here).

Operationally:

1. **Reservation guards.** `SlotReserver::reserve()`, `reserveForBackfill()`, and `reserveForBackfillWithinTransaction()` reject a field whose `is_filterable` is false (typed exception, before any row is touched). No caller may hand a slot to a non-filterable field.
2. **Exhaustion fallback is scoped to filterable fields.** The ADR `0007` enqueue fires iff a *filterable* registered field lacks a live slot. A non-filterable registered field with no slot is the steady state, not a degradation — no queue row, no `capacity_wait`. (`LiveSlotMap` carries `is_filterable` so `PayloadSplitter` can make the distinction.)
3. **Retype of a non-filterable field is registry-only.** `RetypeInitiator` updates `declared_type`, tombstones a legacy live slot if one exists, bumps `stardust_schema_version`, and stops: no replacement reservation, no `backfill_checkpoints` row, nothing for the Reconciler to drain. The JSON payload is authoritative (ADR `0013`), so there is nothing to backfill.
4. **Demotion (`filterable → false`) is tombstone-and-done.** The old slot enters the ADR `0009` eviction lifecycle; reads immediately fall back to `JSON_EXTRACT` — exactly the "seamless demotion" ADR `0013` promises. Promotion (`false → filterable`) is unchanged: the existing Phase 6b path reserves an indexed `backfilling` slot and backfills. A consequence of (3): the `RetypeInitiator` reservation call site only executes for filterable fields, so its `requireIndexed` argument is always true.
5. **Legacy slots are grandfathered.** Pre-existing live slots held by non-filterable fields are not actively migrated; they decay through the eviction lifecycle the next time the field is retyped or otherwise touched. ADR `0033`'s "non-filterable slots are an accepted residual" note shrinks toward zero as this decision takes effect — new deployments never create such slots at all.

## Consequences

**Positive:**

- Slot inventory — the architecture's scarcest, monotonically-growing resource — is spent only where the read path benefits. Every reserved slot is a filter target by construction.
- The infinite `capacity_wait` loop for slot-less non-filterable fields is eliminated; sync-queue depth becomes a true signal of filterable-field backfill debt.
- The write path sheds dead work: no per-page UPSERTs into columns no query will ever read, and non-filterable retypes complete instantly instead of occupying a Reconciler work-source claim per chunk.
- The registry, the ADRs, and the code finally state the same model, closing the trap where the code's *permissiveness* reads as design *intent* (the trap that motivated this review).

**Negative:**

- The "typed retrieval" future option is formally given up: reading a non-filterable field always pays the JSON decode. This was already true in practice — `BoundedFetch` has never served them from slots — so the cost is giving up an unimplemented possibility, not a capability.
- `is_filterable` becomes load-bearing on the write path (the `LiveSlotMap` registry read grows a column and a branch), not just the read path.
- Grandfathered non-filterable slots linger until their field is next touched. On an alpha-stage engine with no production deployments this is a non-issue; a GA-era operator sweep tool can be specified later if real deployments accumulate them.

**Rejected alternatives:**

- **Implement typed retrieval** (make `BoundedFetch` join pages for non-filterable slotted fields) — adds one LEFT JOIN per extra page to avoid decoding a JSON column the fetch already returns; negative-sum on the read side, and it would legitimize spending scarce slot inventory on fields that can never be filter targets.
- **Keep the permissive status quo** — retains dead slot writes, the unserviceable sync-queue loop, and the documented-versus-actual mismatch that misleads every new reader of ADR `0003`.
- **Actively sweep legacy non-filterable slots now** — a data migration with no beneficiary at `0.3.0-alpha.1`; the eviction lifecycle already guarantees eventual decay, and re-run-convergent tooling (ADR `0033`'s pattern) can be added if GA deployments ever need it.

## Related

- ADR `0003` — Schema-Driven Index Provisioning (the withdrawn "typed retrieval" clause; the core schema-driven decision stands)
- ADR `0013` — JSON Payload as System of Record (the Strict Projection Rule this ADR enforces; unindexed fields read via `JSON_EXTRACT`)
- ADR `0007` — Write Availability Over Query Completeness (the backfill queue this ADR scopes to filterable fields)
- ADR `0016` — Field Type Change Lifecycle (the retype tuple this ADR short-circuits for non-filterable fields)
- ADR `0009` — Tombstone-Based Slot Eviction (the decay path for grandfathered and demoted slots)
- ADR `0004` — Fail-Fast on Unindexed Filters (unchanged: non-filterable fields remain rejected as filter targets pre-flight)
- ADR `0031` / `0032` / `0033` — Spread metric, affinity, compaction (their filterable-slots-only population becomes the *entire* slot population under this decision)
