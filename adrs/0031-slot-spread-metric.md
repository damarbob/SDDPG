# 0031 - Slot Spread Metric: Advisory Page-Fragmentation Signal

**Status:** Proposed
**Created:** 2026-06-27

## Context

ADR `0001` chose extension tables over EAV. The structural win is that a query's join cost scales with the number of *distinct extension pages* its referenced fields occupy, not with the number of attributes referenced. A 10-field filter whose fields all live on one page is one `INNER JOIN`; the equivalent EAV query is ten self-joins against a degenerate tall-narrow values table. This is the foundational performance claim of the architecture.

That claim degrades — gently — when a single model's live filterable slots become scattered across multiple pages. We call this **spread**: the number of distinct pages a model's filterable fields occupy. The `SqlFilterCompiler` (Phase 8) emits one `INNER JOIN entry_slots_page_N` per *distinct page* touched, so a model spread across 3 pages pays 2 extra index range-scans per query versus the same model packed onto 1 page. This is a constant-factor regression *inside* the good regime — narrow typed tables with `(tenant_id, slot)` composite indexes — not a return to EAV's structural penalty. But it is real, and it is currently invisible.

Spread is not pathological; it is an emergent consequence of three accepted decisions:

- **ADR `0012`** freezes extension-page DDL. A slot cannot be moved to another page by `ALTER TABLE`; relocation is the full tombstone → reserve → backfill → sweep cycle (ADR `0009`).
- **ADR `0009`** / **ADR `0016`** relocate slots through that eviction lifecycle. `SlotReserver` selects the *globally oldest matching free slot*, with no model affinity — so a field retyped or promoted today can land on a different page than its model-siblings.
- Type-family ceilings (25 `str` / 15 `int` / 10 `num` / 10 `dt` per page) force spread structurally for models that exceed a family's per-page capacity.

The result mirrors the residual hazard ADR `0019` named for cardinality: the architecture has a known second-order cost that no signal surfaces. The schema designer and the operator are told spread *can* happen, but nothing tells them *when it is happening, for which model, and by how much* until it shows up as query latency.

The two policy edges:

- **No signal at all** — operators discover spread only through the slow-query log or by manually inspecting `stardust_slot_assignments`. The "join cost stays bounded" promise of ADR `0001` is unverifiable in production.
- **Automatic compaction on a timer** — a scheduled global pass that re-packs every model onto a minimal page set. This is technically possible but its cost is upside-down: relocating a field is a full re-write of that model's filterable projection (a scan of `entry_data` per relocated field plus a sweep of the old page) and a transient de-indexing window (a `backfilling` field falls back to JSON-payload reads per ADR `0013`). Running that fleet-wide on a timer subjects hot tenants to recurring mass I/O to shave a bounded constant factor off queries that were already fast. Worse, "optimal" is a moving target: co-location (few pages per model) and slot density (few pages globally) are opposed objectives — a re-packer is continuously re-solving a bin-packing tradeoff whose answer drifts with every new field.

The right policy sits between, exactly as ADR `0019`'s does for cardinality: **measure spread asynchronously, surface it as a structured-log advisory event, and let operators act.** Remediation (operator-initiated per-model compaction, or model redesign) is the subject of forthcoming companion ADRs; this ADR defines only the measurement and the signal.

## Decision

A schema-registry advisory pipeline measures per-model spread on a configurable schedule and on demand, and emits structured-log events when a model's spread exceeds its theoretical minimum by a configurable margin. **No spread event ever blocks, rejects, slows, or auto-remediates anything.** The pipeline is purely observational, identical in posture to the cardinality advisory of ADR `0019`.

### Definitions

For a given `(tenant_id, model_id)`:

- **`pages_occupied`** = the count of distinct `page_id`s the model's *live filterable slots* occupy. Live filterable means `stardust_slot_assignments.status IN ('assigned', 'ready')` AND the mapped `stardust_fields.is_filterable = 1`. `backfilling` and `tombstoned` slots are excluded — they are not query-servicing, so they do not contribute join cost.
- **`theoretical_min_pages`** = the fewest pages that could hold the model's filterable fields given the per-family capacities. Because a single page provides all four families simultaneously, the minimum is governed by the most-constrained family:

  ```
  theoretical_min_pages = max over families f of  ceil( filterable_field_count[f] / capacity[f] )
  ```

  where `capacity = { str: 25, int: 15, num: 10, dt: 10 }`. A model with 30 string + 5 int filterable fields has `theoretical_min_pages = max(ceil(30/25), ceil(5/15)) = max(2, 1) = 2`.
- **`excess_pages`** = `pages_occupied - theoretical_min_pages`. This is the count of avoidable extra joins a fully-referencing query pays. `excess_pages = 0` is optimal packing; it is the metric operators act on.

### Sampling Triggers

A spread sample is taken under three conditions:

1. **Periodic.** Each model is re-sampled on the Watcher's existing jittered daily cadence — the same schedule that drives ADR `0019`'s periodic cardinality sample (`$cardinalityIntervalSeconds` / `$cardinalityJitterSeconds`). Spread drifts only on registry mutation, so a daily cadence is generous; reusing the existing schedule avoids a second timer and a second stampede surface.
2. **Post-relocation (one-shot).** Any pipeline that relocates a slot — retype (ADR `0016`), filterability promotion, or the forthcoming compaction tool — issues a one-shot spread sample for the affected `(tenant_id, model_id)` after it commits. This gives operators immediate confirmation of the spread delta a relocation caused (and, for compaction, confirmation that spread actually dropped).
3. **On demand.** An operator CLI (`bin/stardust spread:report [--tenant=N] [--model=N]`) runs the sample synchronously and prints the result, for triage outside the daily window.

The pipeline does NOT sample inside the read or write path. Spread measurement is a registry-only query and never contends with consumer traffic.

### Sampling Method

Spread is computed entirely from the registry — it never touches `entry_data` or any extension page, so it is far cheaper than the cardinality sample of ADR `0019` (which scans the tenant partition):

```sql
SELECT
  f.model_id                         AS model_id,
  COUNT(DISTINCT sa.page_id)         AS pages_occupied,
  COUNT(*)                           AS filterable_slot_count,
  sa.slot_column                                                  -- for the per-family breakdown
FROM stardust_slot_assignments sa
JOIN stardust_fields f ON f.id = sa.field_id
WHERE sa.tenant_id = :tenant_id
  AND sa.status IN ('assigned', 'ready')
  AND f.is_filterable = 1
GROUP BY f.model_id;
```

The per-family counts needed for `theoretical_min_pages` are derived from the `slot_column` prefix (`i_str_`, `i_int_`, `i_num_`, `i_dt_`) in application code — no second query. The whole sample is bounded by the count of a tenant's live filterable slots, which is small (tens to low hundreds), and uses the registry's existing indexes.

### Threshold and Event Emission

For each sampled model the pipeline computes `pages_occupied`, `theoretical_min_pages`, and `excess_pages`.

A `spread_sampled` event is emitted on **every** sample regardless of threshold — the continuous signal for dashboards and trend analysis, mirroring ADR `0019`'s `cardinality_sampled`.

A `high_spread_model` event is additionally emitted when BOTH:

- `pages_occupied >= 2` (a model that legitimately needs a single page is never flagged), AND
- `excess_pages >= spread_excess_page_threshold` (default `2` — i.e. the model occupies at least two more pages than it needs, paying two-or-more avoidable joins).

Both bounds are deployment-tunable. The default deliberately ignores `excess_pages == 1` (one avoidable join is rarely worth a compaction's cost) and only alerts when fragmentation has compounded. Operators running latency-critical models may tighten the threshold to `1`; operators who treat spread as cosmetic may raise it.

### Event Schema

Both events are structured log records on `source: registry` per ADR `0020`, consistent with the cardinality advisory. Required source-specific fields:

| Field                   | Type    | Description                                                            |
| :---------------------- | :------ | :-------------------------------------------------------------------- |
| `tenant_id`             | integer | The partition the sample covered.                                     |
| `model_id`              | integer | The `stardust_models.id` sampled.                                     |
| `pages_occupied`        | integer | Distinct pages the model's live filterable slots occupy.              |
| `theoretical_min_pages` | integer | Fewest pages the filterable fields could occupy (family-constrained). |
| `excess_pages`          | integer | `pages_occupied - theoretical_min_pages`.                             |
| `filterable_slot_count` | integer | Live filterable slots counted for this model.                         |
| `trigger`               | string  | `periodic`, `post_relocation`, or `on_demand`.                        |

`high_spread_model` carries the same fields plus:

| Field       | Type    | Description                                          |
| :---------- | :------ | :--------------------------------------------------- |
| `threshold` | integer | The `spread_excess_page_threshold` in force.         |

### Operator Action

The pipeline emits events; it does not act. When `high_spread_model` fires, the operator's options are:

- **Accept** — leave the model as-is if the extra joins are tolerable for its workload. The default for cosmetic spread.
- **Compact** — relocate the model's filterable slots onto a minimal page set via the forthcoming operator-initiated compaction tool (a same-type retype, reusing the ADR `0016` backfill lifecycle, targeting co-located pages). This is the cure, paid per-model and on the operator's schedule — never automatically.
- **Redesign** — reduce the model's filterable-field footprint, or split a family-saturated model. An operator/data-modeling decision, out of architectural scope.

The architecture deliberately does **not** auto-compact, for the same family of reasons ADR `0019` does not auto-demote: compaction is a mass-I/O migration with a transient de-indexing window (`backfilling` fields read from the JSON payload, ADR `0013`), and its cost is justified only against a specific hot model, not on a blind timer. The signal is the contribution; the remedy is the operator's call.

## Consequences

**Positive:**

- The "join cost stays bounded" promise of ADR `0001` becomes a queryable, alertable signal (`high_spread_model`) instead of an unverifiable claim. Operators can watch spread the same way ADR `0019` lets them watch cardinality.
- The sample is registry-only — no `entry_data` or extension-page scan — making it dramatically cheaper than the cardinality sample and safe to run on every model on the daily cadence indefinitely.
- The pipeline reuses existing infrastructure end to end: the Watcher's jittered schedule, the `source: registry` structured-log channel (ADR `0020`), and the registry tables already maintained for every other concern. The only new code is one bounded query and a min-pages computation.
- `excess_pages` is directly actionable — it is the literal count of avoidable joins, and `0` is the unambiguous target. Operators do not have to interpret a ratio or a heuristic.
- The post-relocation trigger closes the loop: a retype or promotion that worsens spread is visible immediately, and a compaction that fixes it is confirmed immediately.

**Negative:**

- The advisory is best-effort and periodic. A relocation that worsens spread between two periodic samples is invisible for up to a day unless it goes through a pipeline that fires the post-relocation one-shot. Acceptable for spread (which only changes on registry mutation, never on data ingest), less so for tight operational SLOs.
- Spread is measured but never remediated by this ADR. The operator must possess the compaction tool (forthcoming) or accept manual remediation; until that tool ships, `high_spread_model` is actionable only by model redesign.
- `theoretical_min_pages` assumes the standard per-family page layout. If a future ADR introduces heterogeneous page layouts, the min-pages formula must be revised in lockstep, or the metric will misreport excess.
- The metric is per-tenant per-model. A model fragmented for tenant A but compact for tenant B emits independently per partition; operators must read `tenant_id` to triage — the same tenant-awareness ADR `0019` already requires.
- Threshold tuning is deployment-specific. The conservative default (`excess_pages >= 2`) will under-report for latency-critical operators and over-report for spread-tolerant ones; the bound is explicit configuration, not a guess the pipeline makes.

**Rejected alternatives:**

- **Automatic compaction on a timer** — the policy edge this ADR explicitly rejects. A scheduled global re-pack is a recurring dataset-rewrite with a transient de-indexing window, spent to shave a bounded constant factor; and because co-location and slot density are opposed objectives, the "optimal" packing it chases drifts continuously. Maximal cost, bounded benefit, permanent operational liability.
- **Per-query join-count logging instead of registry sampling** — the `search_request` event (Phase 8) could carry a `distinct_page_count`, giving an online spread signal. But it only sees models that are actually queried, samples them unevenly by traffic, and adds a field to the hot-path event. Registry sampling is the authoritative, complete, traffic-independent measurement; a complementary `distinct_page_count` on `search_request` is a reasonable future addition but not a substitute.
- **Dedicated per-model pages to prevent spread structurally** — eliminating spread by giving each model its own pages collapses slot density (a 3-field model squats a 60-slot page) and explodes total page count, worsening the global side of ADR `0012`'s negative #2. Prevention belongs in a reservation *bias* (the forthcoming model-affinity ADR), not in measurement, and never as dedicated pages.
- **`INFORMATION_SCHEMA`-derived metrics** — spread is a registry concept (which model's slots sit on which pages), not a MySQL storage statistic. `INFORMATION_SCHEMA` has no notion of it; the registry is the only source.
- **A single absolute `pages_occupied` threshold (no theoretical minimum)** — would flag a model that genuinely needs 3 pages identically to one that wastefully occupies 3 where 1 suffices. `excess_pages` distinguishes unavoidable spread from avoidable spread; the absolute count cannot.

## Related

- ADR `0001` — Extension Tables Over EAV (the join-cost model that spread regresses; this ADR makes its bound observable)
- ADR `0009` — Tombstone-Based Slot Eviction (the relocation lifecycle that both causes and cures spread)
- ADR `0012` — Immutable Extension Page DDL (why a slot cannot be cheaply moved; the root of monotonic-page-growth negative #2)
- ADR `0013` — JSON Payload as System of Record (the read fallback during a compaction's `backfilling` window)
- ADR `0014` — Schema-Level Safety Over Runtime Circuit Breaking (the advisory-not-enforcement posture this ADR inherits)
- ADR `0016` — Field Type Change Lifecycle (a spread driver; fires the post-relocation one-shot)
- ADR `0017` — Schema Registry as Coordination Contract (the tables sampled)
- ADR `0019` — Index Cardinality Policy (the sibling advisory; shares the Watcher cadence, the `source: registry` channel, and the never-auto-remediate principle)
- ADR `0020` — Structured Logging Mandate (the `spread_sampled` / `high_spread_model` event vocabulary)
