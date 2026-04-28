# 0019 - Index Cardinality Policy: Asynchronous Advisory, Never Runtime Enforcement

**Status:** Proposed
**Created:** 2026-04-22

## Context

ADR `0014` establishes that StarDust does not implement runtime circuit breakers, scanned-row tracking, or query-time abort mechanisms. Query safety is enforced exclusively at the schema level via pre-flight rejection (ADR `0004`) and bounded page size (ADR `0005`). That decision deliberately accepts a residual hazard: a tenant who provisions an index on a low-cardinality field — a boolean with 50/50 distribution, an enum with two practical values, a "deleted" flag where 99% of rows share one value — passes pre-flight rejection (the slot has `is_filterable = true`) but causes the optimizer to scan a large fraction of the dataset to fulfill `LIMIT page_size + 1`. The query is bounded but slow, and the framework does not detect or warn about it.

ADR `0014` "Negative" is explicit: "The system does not self-protect against poorly designed schemas." That position is correct as a decision about *runtime* behavior, but it leaves an unresolved operator question: when a low-selectivity index is provisioned, how does the operator find out *before* it becomes a production incident? The schema designer is named as responsible for cardinality, but the architecture provides no signal to the schema designer or the operator monitoring the system.

The two failure modes at the policy edges are:

- **Strict runtime enforcement** (reject queries on low-cardinality indexes) — re-introduces the unpredictable API behavior ADR `0014` rejects. Cardinality changes over time; today's "good" index becomes tomorrow's "bad" one as data ingests, and consumers cannot predict acceptance.
- **No signal at all** — operators discover low-cardinality indexes only through the slow-query log, performance dashboards, or end-user complaints. The architectural guarantee that "schema design quality is surfaced as an explicit concern" (ADR `0014` Positive #4) is aspirational rather than realized — nothing actually surfaces it.

The right policy lives between these: cardinality is measured asynchronously, surfaced as a structured-log advisory event, and acted upon by operators (typically by demoting `is_filterable` and triggering the eviction lifecycle of ADR `0009`). It is never used to gate query execution.

## Decision

The schema-registry advisory pipeline samples cardinality on each indexed slot on a configurable schedule and at well-defined lifecycle events, and emits structured-log events when measured selectivity falls below a configurable threshold. **No advisory event ever blocks, rejects, or aborts a query.** The pipeline is purely observational.

### Sampling Triggers

A cardinality sample is taken on a slot under either condition:

1. **Post-backfill (one-shot).** When the Reconciler advances a `stardust_slot_assignments` row from `backfilling → ready` (per ADR `0016`'s promote step), it issues a cardinality sample on the newly-ready slot in the same logical operation as the promotion. This is the earliest moment the slot has full data, and the sample becomes the slot's baseline reading. This trigger fires for every newly-promoted slot — type changes, filterability promotions, and freshly-assigned cold slots all run through `backfilling → ready` and therefore all sample at promotion.
2. **Periodic re-sample.** Each indexed slot is re-sampled on a configurable schedule (default: every 24 hours, jittered to avoid stampedes). Cardinality drifts as data ingests; a slot that was selective at promotion can become unselective months later, and operators need to know.

The pipeline does NOT sample synchronously inside the read path or the write path. Both triggers fire on the same lazy cadence as the existing daemons; cardinality measurement does not contend with consumer traffic.

### Sampling Method

For each indexed slot, the pipeline executes:

```sql
SELECT
  COUNT(*)                              AS row_count,
  COUNT(DISTINCT <slot_column>)         AS distinct_values
FROM <entry_slots_page_X>
WHERE tenant_id = <tenant_id>;
```

The query is bounded by the per-tenant partition (the same tenant scoping as the read path) and uses the slot's existing covering index. On large pages, the query is allowed to use `EXPLAIN ANALYZE` for `distinct_values` estimation rather than an exact count — exact precision is not required; order-of-magnitude is.

`MySQL 8.0.13` (per ADR `0023`) is the floor for this pipeline because `EXPLAIN ANALYZE` and the registry's other supporting features rely on it. No older-version fallback is supported.

### Threshold and Event Emission

For each sampled slot, the pipeline computes:

- **Selectivity** = `distinct_values / row_count` (clamped to `[0, 1]`).
- **Distinct floor** = `distinct_values`.

A `low_cardinality_index` event is emitted when EITHER condition is met:

- Selectivity < `0.01` (the slot's index covers fewer than 1 distinct value per 100 rows on average), AND
- `row_count >= 10,000` (small tables are not flagged — selectivity is undefined for tiny populations).

OR

- `distinct_values < 10` regardless of selectivity (an index with under 10 distinct values is structurally low-cardinality even on small populations; this catches the "boolean / two-state enum" case before the table grows).

Both thresholds are deployment-tunable via configuration. The defaults reflect "indexes with worse than 1% selectivity are advisory-flagged" — a deliberately conservative bar that catches obvious traps (booleans, soft-delete flags, low-cardinality status enums) without crying wolf on legitimate moderate-cardinality fields.

A `cardinality_sampled` event is emitted on **every** sample, regardless of whether the threshold was crossed. This gives operators a continuous signal for dashboards and trend analysis; the threshold-crossing `low_cardinality_index` event is for alerting.

### Event Schema

Both events are structured log records on `source: registry` per ADR `0020`. Required source-specific fields:

| Field             | Type     | Description                                                          |
| :---------------- | :------- | :------------------------------------------------------------------- |
| `slot_assignment_id` | integer | The `stardust_slot_assignments.id` whose slot was sampled.        |
| `field_id`        | integer  | The mapped `stardust_fields.id`.                                     |
| `tenant_id`       | integer  | The partition the sample covered.                                    |
| `page_id`         | integer  | The `stardust_pages.id` (denormalized for triage).                   |
| `slot_column`     | string   | E.g., `i_str_03`.                                                    |
| `row_count`       | integer  | Sampled population size.                                             |
| `distinct_values` | integer  | Estimated distinct values (exact for `COUNT(DISTINCT)`, estimated for `EXPLAIN ANALYZE`). |
| `selectivity`     | float    | `distinct_values / row_count`, rounded to 4 decimal places.          |
| `trigger`         | string   | `post_backfill` or `periodic`.                                       |

A `low_cardinality_index` event additionally includes:

| Field             | Type     | Description                                                          |
| :---------------- | :------- | :------------------------------------------------------------------- |
| `threshold_violated` | string | `selectivity` or `distinct_floor` (which rule fired; both can fire and the event lists both). |

### Operator Action

The advisory pipeline only emits events; it does not act. When a `low_cardinality_index` event fires, the operator's options are:

- **Accept** — leave the index in place if the slow scan is tolerable for this workload.
- **Demote** — set the field's `is_filterable = false`. The standard sever-tombstone-sweep lifecycle (ADR `0009`) reclaims the slot. Filters on this field will then be rejected with `400 Bad Request` per ADR `0004`.
- **Restructure** — split the field into multiple higher-cardinality fields, or model the data differently. Out of architectural scope; an operator/data-modeling decision.

The architecture deliberately does not auto-demote. Auto-demotion would re-introduce unpredictable API behavior — a previously-accepted filter starts returning `400` because cardinality drifted. ADR `0014`'s deterministic-API guarantee covers `is_filterable` flips just as it covers query acceptance: the flag changes only by explicit operator action.

## Consequences

**Positive:**

- The "schema design quality is surfaced as an explicit concern" claim from ADR `0014` is realized as a concrete signal (`low_cardinality_index`) rather than left as a pious wish. Operators have a queryable, alertable surface for the cardinality residual hazard.
- API determinism (ADR `0014`) is preserved. No query is ever rejected, aborted, or slowed by the advisory pipeline. Pre-flight rejection's behavior is unchanged.
- The pipeline reuses existing infrastructure. Sampling is a bounded `SELECT` against the slot's covering index (no extra schema), event emission is the structured-log mandate that already exists for every other daemon (per ADR `0020`), and the post-backfill trigger piggybacks on the Reconciler's existing promote step.
- The dual threshold (selectivity AND distinct floor) catches both "high-volume low-selectivity" indexes and "low-cardinality from inception" indexes with one rule set. Operators do not need to write per-field overrides for boolean-flag fields.
- `cardinality_sampled` provides a continuous signal independent of whether the threshold was crossed. Long-term cardinality drift is visible in dashboards before it becomes alertable.
- The advisory's emission cadence (post-backfill + 24h periodic) is light enough to run forever in production. Sampling cost is bounded by tenant partition size and the existing covering index.

**Negative:**

- The advisory is best-effort. A slot whose cardinality crosses the threshold between two periodic samples is invisible to the pipeline for up to 24 hours. For most operator workflows this is acceptable; for tight SLOs it is not.
- `EXPLAIN ANALYZE`-based estimates are approximate. Operators reviewing dashboards should treat cardinality numbers as order-of-magnitude, not exact. Exact `COUNT(DISTINCT)` is available for spot checks but is too expensive to run on every sample for every slot.
- Cardinality is measured per tenant partition. A field that is highly selective for tenant A and a 50/50 boolean for tenant B emits `low_cardinality_index` for tenant B's partition only — operators must read the `tenant_id` field to triage. This is a feature, not a bug (per-tenant index quality is the right granularity), but it does require operators to be tenant-aware.
- Threshold tuning is deployment-specific. Defaults are conservative; operators of OLTP-heavy workloads may want tighter bars, while operators with deliberately low-cardinality reporting indexes may want looser bars. The configuration surface is explicit.
- The pipeline adds a periodic load to MySQL proportional to the count of indexed slots times the per-tenant population size. On very large deployments this is non-trivial — tunable via `cardinality_sample_interval` and `cardinality_sample_jitter` to spread it out.

**Rejected alternatives:**

- **Reject queries on low-cardinality indexes at runtime** — directly contradicts ADR `0014`. Re-introduces the unpredictable API behavior and the "succeeds today, fails tomorrow" failure mode. The case was already litigated.
- **Only sample at provision time** — provision-time cardinality is zero (the page is empty). Useless. The post-backfill trigger is the earliest moment with meaningful data.
- **Continuous cardinality counters maintained on every write** — adds a counter update to every write, multiplying write contention on hot fields. The asynchronous sample model achieves the same advisory signal without contaminating the write hot path.
- **Auto-demote the field's `is_filterable` flag when below threshold** — silently flips API behavior for that field. A filter that worked yesterday starts returning `400` today. Re-introduces unpredictability for the same reason ADR `0014` rejected runtime breakers.
- **Use `INFORMATION_SCHEMA` cardinality estimates only** — MySQL's `INFORMATION_SCHEMA.STATISTICS.CARDINALITY` is a per-index heuristic, not per-tenant-partition. The cardinality signal we want is "how selective is this index *for this tenant's data*"; the MySQL-level estimate aggregates across tenants and is therefore the wrong granularity.
- **Single threshold (selectivity only)** — misses the boolean-on-tiny-table case. A 100-row table with 50/50 split has selectivity 0.02 (above the 0.01 default), but the index is structurally useless. The distinct-floor rule catches it.
- **Single threshold (distinct count only)** — misses the high-volume-low-selectivity case. A 10M-row table with 50 distinct values has `distinct_values = 50` (above any reasonable distinct floor) but selectivity 0.000005 — a clear advisory candidate. The selectivity rule catches it.

## Related

- ADR `0004` — Fail-fast on unindexed filters (advisory does NOT change pre-flight behavior)
- ADR `0009` — Tombstone-Based Slot Eviction (the operator's "demote" path uses this lifecycle)
- ADR `0014` — Schema-Level Safety Over Runtime Circuit Breaking (this ADR closes the residual hazard ADR `0014` named without contradicting it)
- ADR `0016` — Field Type Change Lifecycle (post-backfill trigger fires at the promote step defined here)
- ADR `0017` — Schema Registry as Coordination Contract (the registry tables sampled by the pipeline)
