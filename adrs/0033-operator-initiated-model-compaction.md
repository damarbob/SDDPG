# 0033 - Operator-Initiated Model Compaction

**Status:** Proposed
**Created:** 2026-07-04

## Context

ADR `0031` measures **spread** — the number of distinct extension pages a model's live filterable slots occupy — and emits `high_spread_model` when a model's `excess_pages` crosses the operator's threshold. ADR `0032` biases new reservations toward co-location so fresh and incrementally-grown models stay compact. Neither converges a model that is *already* fragmented: the metric only observes, and affinity is forward-only. Something must be the cure, and under ADR `0012` (no `ALTER TABLE` on populated pages) the only mechanism for moving a slot's contents to another page is the relocation lifecycle already defined by ADR `0009` and ADR `0016`: tombstone the old slot, reserve a new one, backfill it from the JSON payload, and let the Liberator sweep the vacated column.

The policy edges were already litigated in ADR `0031`:

- **Scheduled or automatic compaction** — a recurring, fleet-wide dataset-rewrite with a transient de-indexing window, spent to shave a bounded constant factor, continuously re-solving the opposed co-location/density bin-pack. Rejected there; that rejection stands.
- **No tool at all** — leaves `high_spread_model` actionable only by model redesign. The signal names a problem the operator cannot surgically fix.

The middle is a **deliberate, operator-initiated, per-model operation**: when the metric flags a specific hot, fragmented model, the operator pays the relocation cost exactly once, exactly there.

The decisive implementation fact — verified against the shipped Phase 6b pipeline — is that compaction is almost entirely already built. The ADR `0024` coercion matrix short-circuits its **identity diagonal** (`source type === target type` → the value passes through, normalised), so a *same-type retype* is mechanically nothing but a slot relocation with a data copy. The retype initiation transaction does not reject same-type initiations, the checkpoint contract (`backfill_checkpoints` rows named `retype_field_{id}` with `source_declared_type`) is exactly reusable, and the Reconciler's `RetypeBackfillWorkSource` will drain such a checkpoint **unmodified** — chunked identity backfill, `backfilling → ready` promotion, schema-version bump, the ADR `0019` post-backfill cardinality sample, and ADR `0031`'s `trigger='post_relocation'` spread sample all fire per existing spec. What is genuinely missing is only: a planner that chooses target pages, a reservation variant that can *pin* the chosen page, and an orchestration surface.

Two behaviours of the existing lifecycle shape the design and must be stated up front:

- **Filters on a `backfilling` field are rejected** (ADR `0004` / ADR `0016`): during each field's relocation window, filtered queries on it fail with a typed exception and reads fall back to the JSON payload (ADR `0013`). Compaction inherits this verbatim — relocating K fields simultaneously would reject filters on all K at once.
- **Double-occupancy**: a relocated field holds two slots — the old (`tombstoned`, awaiting Liberator sweep per ADR `0009`) and the new (`backfilling`) — until reclamation completes. Compaction consumes free capacity on its target pages *before* returning any, precisely when the model's pages are fragmented.

## Decision

StarDust gains an **operator-initiated model compaction** operation that relocates a model's live filterable slots onto a minimal page set, implemented as a sequence of **same-type retypes** riding the unmodified Phase 6b backfill pipeline. It is never scheduled, never daemon-triggered, and never automatic.

### Public surface

`StarDust::compactModel(int $tenantId, int $modelId, …options)` plus a CLI:

```
bin/stardust compact:model --tenant=N --model=N [--dry-run] [--parallel=N]
```

Tenant/model validation follows the existing tenant-scoped pattern (a model that does not exist or belongs to another tenant is indistinguishable — same posture as `ExportJobSubmitter::getJob()` and `RetypeInitiator`).

### The planner

A pure registry computation — no data-plane reads, no locks held across planning:

1. **Population**: the model's **live filterable slots** (`status IN ('assigned','ready')` joined to `stardust_fields.is_filterable = 1`) — deliberately the *same population as ADR `0031`'s metric*, so the metric that triggers compaction and the operation that cures it measure the same thing, and `excess_pages → 0` is the verifiable success criterion. Non-filterable slots are out of scope (see Consequences).
2. **Plan**: compute `pages_occupied` and `theoretical_min_pages` (ADR `0031`'s family-ceiling formula), then select a **minimal target page set**, preferring pages that already host the model's slots and have the most per-family free capacity. Fields whose live slot already sits on a target page are **no-ops** — compaction relocates only the delta. This makes re-running convergent and cheap.
3. **Up-front capacity check**: the plan is admissible only if the target pages hold enough `free` slots of each required family for every relocation, accounting for double-occupancy (vacated slots return only after Liberator sweep and therefore count as unavailable). If insufficient, the operation fails fast with a typed `CompactionCapacityException` before **any** mutation — no partial initiation. The operator's remedies are to wait for Liberator reclamation, let the Watcher provision capacity, and re-run.
4. **`--dry-run`** prints the full plan — current spread, chosen target pages, per-field relocations, no-op count, capacity verdict — and exits without mutating anything or emitting events.

### Per-field relocation is a same-type retype

Each relocation runs the ADR `0016` initiation tuple in one transaction, exactly as the retype initiator does, with the declared type and filterability **unchanged** (no `stardust_fields` mutation): tombstone the field's current live slot; reserve a new `backfilling` slot **on the planner's pinned target page** (indexed, per ADR `0003`); bump `stardust_schema_version`; insert a standard `retype_field_{id}` checkpoint with `source_declared_type` set to the field's current type — which routes the backfill through the ADR `0024` **identity diagonal**.

**No deferred reservation.** This is the one deliberate divergence from ADR `0016` commitment 4 (retype defers when no free slot exists and lets the work source retry later). A deferred compaction reservation would eventually land on whatever page the reserver picks — not the planner's choice — defeating the operation's entire purpose. Compaction therefore **pins or fails**: the reservation happens on the planned page inside the initiation transaction, or the initiation does not happen. The divergence is safe because admissibility was verified at plan time and the failure mode is a clean typed exception, not a stuck half-migration.

This requires one new `SlotReserver` variant — a page-pinned sibling of the existing within-transaction reservation (illustratively `reserveForBackfillOnPageWithinTransaction(fieldId, pageId, requireIndexed: true)`), composing the existing indexed-candidate filter with a `page_id` predicate. No existing signature changes.

### Sequential orchestration

By default the CLI is a **long-lived operator process** that relocates **one field at a time**: initiate field *k*, poll its checkpoint until the Reconciler's unmodified `RetypeBackfillWorkSource` marks it `completed` (the promotion to `ready` restores the field's filterability), then initiate field *k+1*. At any moment, **exactly one** of the model's fields has filters rejected — the operator-facing degradation is bounded and predictable. `--parallel=N` raises the in-flight count for operators who prefer wall-clock speed over filter availability; per-field overlap remains guarded by the existing `RetypeInProgressException` checkpoint uniqueness.

**Resume is re-run.** If the CLI crashes, nothing is stuck: in-flight checkpoints are standard retype checkpoints and the Reconciler drains them to completion regardless. Re-running `compact:model` recomputes the remaining delta — already-relocated fields sit on target pages and are no-ops — so the operation converges idempotently with **no compaction-specific state table, no new daemon, and no new work source**.

### Backfill execution and verification

The chunked backfill, `backfilling → ready` promotion, schema-version bumps, ADR `0019` post-backfill cardinality sample, and ADR `0031` `trigger='post_relocation'` spread sample are all the existing pipeline, untouched. The final field's spread sample doubles as the operation's built-in success confirmation: `excess_pages` at (or near, if family ceilings force >1 page) zero.

### Events

Two new structured-log events on `source: registry` (the ADR `0020` allowlist gains them at implementation time): `compaction_planned`, emitted once when a real (non-dry-run) run commits to a plan — carrying `tenant_id`, `model_id`, `pages_before`, `target_pages`, `fields_to_relocate`, `noop_count` — and `compaction_complete`, emitted when the last field promotes — carrying `fields_relocated`, `pages_before`, `pages_after`, `duration_seconds`. Per-field lifecycle events reuse the existing `retype_started` / `chunk_complete` / `promote_to_ready` vocabulary unchanged. Dry-run prints to stdout only and emits nothing, honouring the discipline that events never describe uncommitted state.

## Consequences

**Positive:**

- The cure is a **thin orchestration over shipped machinery**: a registry-only planner, one page-pinned reservation variant, and a CLI loop. The backfill work source, coercion engine, checkpoint contract, promotion path, and both advisory samplers are reused byte-for-byte — the highest-risk parts of a data migration are the parts that already have smoke coverage.
- Cost is paid **per model, on the operator's schedule, only where the metric justifies it** — the exact posture ADR `0031` committed to when it rejected scheduled global re-packing.
- The sequential default bounds degradation to **one field's filters at a time**, and even that field's reads keep working via JSON fallback (ADR `0013`). There is no moment when the model as a whole is unqueryable.
- **Crash-safe by construction**: an interrupted run leaves only standard retype checkpoints that drain to completion, and a re-run converges on the remaining delta. No orphaned state, no operator cleanup.
- Success is **machine-verifiable**: the post-relocation spread sample closes the loop the same run opened, and `spread_sampled` dashboards show the before/after.
- Together with ADR `0032`, the economics invert: affinity keeps new spread rare, so compaction stays an exceptional event rather than routine maintenance.

**Negative:**

- Filters on the in-flight field are rejected for its backfill window, and the sequential default stretches total wall-clock time to roughly K windows for K relocated fields. Operators compacting during low-traffic hours is the expected pattern; `--parallel=N` trades availability for speed but never removes the window.
- Double-occupancy means compaction **needs free headroom exactly when pages are fragmented**. A badly exhausted deployment may have to wait on Liberator sweeps or Watcher provisioning before a plan becomes admissible — the capacity check makes this explicit rather than discovering it mid-flight.
- The long-lived CLI is an operational commitment (a terminal, a tmux session, a maintenance window). The crash behaviour is benign, but the operator owns the process lifetime.
- Non-filterable slots are not relocated. This costs nothing: the read path never consults them (`FieldDescriptor::isIndexedNow()` requires filterability, so `BoundedFetch` never joins their pages — such fields are always served from the JSON payload per ADR `0013`), and under ADR `0034` non-filterable fields are JSON-only, making any live slot they still hold a grandfathered legacy artifact that the eviction lifecycle decays on its own. Excluding them from compaction is therefore correct by construction, not a residual trade-off.
- An identity backfill can still surface `coercion_null` rows: a stored payload value that no longer parses under its own declared type (malformed legacy data) normalises to `NULL` in the new slot, with the standard audit event. This is visibility, not loss — the JSON payload remains authoritative (ADR `0013`) — but operators should expect nonzero `coercion_null` counts on dirty legacy models.

**Rejected alternatives:**

- **Scheduled or automatic compaction** — re-litigated and re-rejected; see ADR `0031`'s rejected alternatives. The trigger is an operator reading a metric, never a timer.
- **All-at-once initiation as the default** — initiating every relocation up front rejects filters on all K fields simultaneously for the whole drain. Offered behind `--parallel=N` for operators who explicitly choose it; never the default.
- **A compaction daemon or persisted plan table** — a fourth daemon and new schema to solve a problem re-run-convergence already solves. The checkpoint rows *are* the durable state; adding more violates the thin-orchestration principle and ADR `0008`'s deliberate daemon census.
- **Deferred reservations (verbatim ADR `0016` semantics)** — the work source's later retry reserves on a page the planner did not choose, silently producing a compaction that does not compact. Pin-or-fail is the only shape that preserves the operation's meaning.
- **`ALTER TABLE`-based moves** — forbidden on populated pages by ADR `0012`; the entire lifecycle exists because of that prohibition.
- **Including non-filterable slots** — widens the cure's scope beyond the metric that triggers it, breaking the "excess_pages → 0 verifies success" contract, for zero query-side benefit: the read path never consults non-filterable slots, and ADR `0034` makes them JSON-only legacy artifacts the eviction lifecycle already decays.

## Related

- ADR `0031` — Slot Spread Metric (the trigger and the built-in verification; this ADR is the "cure" it deferred)
- ADR `0032` — Model-Affine Slot Reservation (the prevention counterpart; affinity keeps compaction rare)
- ADR `0016` — Field Type Change Lifecycle (the reused lifecycle; the single divergence is pin-or-fail vs deferred reservation)
- ADR `0024` — Type Coercion Matrix (the identity diagonal that makes same-type retype a pure relocation)
- ADR `0009` — Tombstone-Based Slot Eviction (sweep of the vacated slots; the source of double-occupancy)
- ADR `0012` — Immutable Extension Page DDL (why relocation, not `ALTER`, is the only move)
- ADR `0013` — JSON Payload as System of Record (reads stay correct throughout the window)
- ADR `0004` — Fail-Fast on Unindexed Filters (the per-field filter rejection during backfill)
- ADR `0007` — Write Availability Over Query Completeness (the fail-fast capacity check never squats capacity on an inadmissible plan)
- ADR `0019` — Index Cardinality Policy (post-promotion cardinality sample fires per existing spec)
- ADR `0020` — Structured Logging Mandate (`compaction_planned` / `compaction_complete` join the registry-source vocabulary at implementation)
- ADR `0034` — Non-Filterable Fields Are JSON-Only (why excluding non-filterable slots from compaction costs nothing; their legacy slots decay via eviction)
