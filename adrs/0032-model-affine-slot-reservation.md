# 0032 - Model-Affine Slot Reservation

**Status:** Proposed
**Created:** 2026-06-27

## Context

ADR `0031` measures **spread** — the number of distinct extension pages a single model's live filterable slots occupy — and surfaces it as an advisory `excess_pages` signal. That ADR is observation; it does not change how slots are assigned. This ADR addresses the other half: **preventing spread at the moment it is created.**

Spread is seeded at reservation time. `SlotReserver` selects the *globally oldest* free slot of the matching type family:

```sql
SELECT a.id, a.page_id, a.slot_column
FROM stardust_slot_assignments a
WHERE a.status = 'free' AND a.slot_type = ?
ORDER BY a.page_id, a.id
LIMIT 1 FOR UPDATE;
```

This ordering has no awareness of which *model* the field being reserved belongs to. As a model's fields are reserved incrementally over time — or relocated through the eviction lifecycle (ADR `0009`) on every retype and filterability promotion (ADR `0016`) — each reservation lands on whichever page happened to have the oldest free slot of the right family. A model's filterable slots therefore drift across pages, and the query compiler (Phase 8) pays one `INNER JOIN entry_slots_page_N` per distinct page touched. This is the root mechanism of spread and of ADR `0012`'s "monotonic page growth" negative #2.

The current global-oldest ordering is not arbitrary: it optimizes **slot density** — packing assignments onto the oldest pages keeps total page count down and keeps the Liberator's sweep surface compact. But density and **co-location** (keeping one model's slots on few pages) are opposed objectives, as ADR `0031` notes. Pushed to its extreme, density maximizes spread; pushed to its extreme, co-location (dedicated per-model pages) collapses density and explodes page count. The question this ADR answers: can reservation reduce spread-at-creation without abandoning density or write availability?

The relevant structural facts:

- `stardust_slot_assignments` rows carry **no `tenant_id` or `model_id`** — only `page_id`, `slot_column`, `slot_type`, `field_id`, `status`. A free slot has `field_id = NULL`. Pages and slots are a **global shared pool**; a slot acquires a tenant/model only when assigned to a field, reached via `field_id → stardust_fields.model_id → stardust_models.tenant_id`. "A model's slots" is therefore a registry join, not a column predicate.
- All three reservation entry points — `reserve()`, `reserveForBackfill()`, `reserveForBackfillWithinTransaction()` — funnel through a single private chokepoint that resolves the field's `declared_type` and runs the candidate `SELECT … FOR UPDATE`. One method changes.

## Decision

Slot reservation becomes **model-affine**: it biases candidate selection toward free slots on pages that already host a **live** slot of the **same model**, and falls back to the existing global-oldest-free ordering when no affine candidate of the required type family exists.

### Affinity, defined

A page is *affine* to the field being reserved when it already hosts at least one `stardust_slot_assignments` row whose status is live (`assigned | backfilling | ready`) and whose `field_id` maps to the **same `model_id`** as the field being reserved. Because slot rows carry no `model_id`, affinity is computed from the registry:

```
page P is affine to field F  ⇔  ∃ live slot on P whose field belongs to model_of(F)
```

where `model_of(F)` is resolved from `stardust_fields.model_id` in the **same query that already resolves `F`'s `declared_type`** — so no new public method parameter is introduced. The `reserve*()` signatures are unchanged (append-only discipline); the model is derived, not passed.

### A bias, not a reservation

This is the load-bearing constraint. Affinity reorders candidates; it never restricts them:

- A model **never owns or reserves whole pages.** An affine page's remaining free slots stay fully available to every other model and tenant under the same global pool.
- When an affine page has **no free slot of the required family**, reservation **spills to global-oldest-free exactly as today.** Affinity can never fail a reservation that would otherwise succeed, so write availability (ADR `0007`) and reservation liveness are preserved bit-for-bit.

The conceptual candidate ordering becomes:

```sql
ORDER BY <page hosts a live slot of this model> DESC,  -- affine pages first
         a.page_id ASC,                                -- then oldest page (density)
         a.id ASC                                      -- deterministic tiebreak
LIMIT 1 FOR UPDATE
```

This composes unchanged with the existing `slot_type` family filter and with the Phase 6b `requireIndexed` `EXISTS`-over-`information_schema.STATISTICS` filter — affinity narrows *ordering*, those clauses narrow *eligibility*, and they stack (an affine **and** indexed slot is preferred, falling back to indexed-anywhere, then nothing). Ordering remains fully deterministic; no randomness is introduced.

### Non-locking affinity (normative)

The candidate `SELECT` runs `FOR UPDATE`. The affinity computation **must not** acquire row locks on the model's live sibling slots. A naïve inline correlated `EXISTS` over `stardust_slot_assignments` inside a `FOR UPDATE` statement can, depending on plan and isolation level, lock the very rows it reads — i.e. the model's in-use slots, which are concurrently read by the write path and mutated by relocations. That contention does not exist today and must not be introduced.

The implementation therefore resolves the affine page-set in a **prior, non-locking read** (e.g. a plain `SELECT DISTINCT page_id` of the model's live slots, or a derived set the planner joins without locking), and only the final free-slot selection takes `FOR UPDATE`, scoped to free rows. The exact SQL shape is an implementation concern; this ADR fixes the policy and the non-locking invariant.

### What this is explicitly not

Affinity is **not** dedicated per-model pages. Giving each model its own pages would let a three-field model squat a sixty-slot page, collapsing density and growing total page count — worsening ADR `0012` negative #2 on the global axis to fix it on the per-query axis. Affinity is the midpoint of the co-location/density line, deliberately nudged toward co-location because spread is the named cost; it buys that nudge only by reordering within the *existing* free-slot pool, never by holding capacity.

## Consequences

**Positive:**

- Models stay compact **by construction**. Freshly created and incrementally grown models keep their filterable slots co-located, so filtered reads touch fewer pages and pay fewer joins — directly lowering ADR `0031`'s `excess_pages` at its source rather than after the fact.
- It reduces how often the (expensive) operator-initiated compaction tool is needed. Prevention is cheap; the cure is a dataset-rewriting migration. Every model affinity keeps compact is a compaction not run.
- **No public API change.** The model is derived from the field inside the existing chokepoint; the three `reserve*()` signatures are untouched, and every current caller (write path, Watcher, retype initiator) is unaffected.
- It reuses the existing `slot_reserved` structured-log event (no new event name, so the `EventVocabularyTest` allowlist is unchanged). An additive payload field — `affinity: 'co_located' | 'fallback'` — can be emitted for observability without touching the event vocabulary.
- Reservation remains deterministic and reproducible: affinity is a stable ordering key, not a heuristic or a random draw.

**Negative:**

- Affinity is **greedy and forward-only.** It lowers the *rate* of new spread but does **not** converge a model that is already spread — that remains the job of operator-initiated compaction. A model fragmented before this policy ships stays fragmented until compacted.
- It **cannot beat structural limits.** A model with more filterable fields of one family than a page holds (e.g. 30 string fields against the 25-`str` ceiling) is spread by necessity, and affinity cannot help. Likewise, on a shared page whose free slots are consumed by other tenants, affinity loses to availability and spills to global-oldest.
- A **marginal density cost.** Preferring an affine (possibly newer) page over an older page with a free slot can leave older free slots unused slightly longer, so total page count may grow marginally faster. This is bounded: affinity only reorders within the *existing* free-slot pool and never provisions a page the Watcher's capacity trigger would not already provision. The trade is a small density give for a meaningful spread-prevention get.
- One additional bounded registry read per reservation (the affine page-set lookup). Cheap — it is registry-only and scoped to the model's live slots — but non-zero.

**Rejected alternatives:**

- **Dedicated per-model pages** — eliminates spread structurally by giving each model its own pages, but collapses slot density (small models squat whole pages) and explodes total page count, worsening ADR `0012` negative #2 globally. Prevention belongs in an ordering *bias*, never in capacity ownership.
- **Affinity as a hard constraint** (reserve only on an affine page, else capacity-wait or fail) — would break write availability (ADR `0007`) and could starve a reservation indefinitely when no affine free slot of the family exists. Bias-with-fallback is the only form that preserves liveness.
- **Denormalizing `model_id` onto `stardust_slot_assignments`** as an indexed column — would make affinity a cheap single-table indexed lookup, but adds a column that must be kept in sync on every `free → assigned → tombstoned → free` transition and every relocation. Given the free-slot candidate set per family is small, the registry join is sufficient; the denormalization is **deferred** as a possible future performance ADR, not adopted here.
- **Relying on compaction alone** — purely reactive. Affinity is the cheap, continuous prevention that keeps the compaction workload small; omitting it makes the costly cure the only lever.

## Related

- ADR `0031` — Slot Spread Metric (this ADR reduces the metric at its source; the metric verifies affinity's effect over time)
- ADR `0012` — Immutable Extension Page DDL (why a scattered slot cannot be cheaply moved; the root of monotonic-page-growth negative #2 that spread aggravates)
- ADR `0009` — Tombstone-Based Slot Eviction (the relocation lifecycle whose re-reservations also flow through affinity)
- ADR `0016` — Field Type Change Lifecycle (retype and filterability promotion both re-reserve; affinity applies to them)
- ADR `0003` — Schema-Driven Index Provisioning (affinity composes with the `requireIndexed` / `is_filterable` eligibility filter)
- ADR `0007` — Write Availability Over Query Completeness (why affinity must be a bias with fallback, never a hard constraint)
- ADR `0017` — Schema Registry as Coordination Contract (the registry join that computes affinity)
- ADR `0029` — Liberator Sweep Omits Tenant Predicate (sweep semantics are unchanged by affinity; co-location does not alter per-page reclamation)
- A forthcoming companion ADR on operator-initiated model compaction will define the *cure* for already-spread models that affinity, being forward-only, cannot converge.
