# Runbook — Maintaining Low Spread

> **Audience:** operators of a running StarDust deployment.
> **Status:** implemented as of 2026-08-19. The spread metric ([ADR 0031](../adrs/0031-slot-spread-metric.md)), model-affine reservation ([ADR 0032](../adrs/0032-model-affine-slot-reservation.md)) and model compaction ([ADR 0033](../adrs/0033-operator-initiated-model-compaction.md)) have all shipped, and all three ADRs are **Accepted**. Every command and event below works today, with one exception called out in §6.2: **`--parallel=N` is not implemented.** The [manual measurement SQL](#33-manual-measurement-today) remains valid.
> On any conflict between this runbook and an ADR, **the ADR governs**.

---

## 1. What spread is and why you care

[Spread](../glossary.md#spread) is the number of distinct extension pages a single model's live filterable slots occupy. Every filtered query pays one `INNER JOIN entry_slots_page_X` per distinct page it references — so a model whose filterable fields sit on three pages pays two extra index range-scans compared to the same model packed onto one.

**Calibrate your urgency correctly.** Spread is a *constant-factor* cost, not a structural cliff: even a badly spread model is paying a couple of extra bounded index scans against narrow, typed, composite-indexed tables. This is nothing like EAV join degradation. You care about spread when a **hot, latency-critical** model accumulates avoidable joins — not as a number to zero out fleet-wide on principle.

The three levers, in the order you should reach for them:

| Lever | What it is | When |
| :--- | :--- | :--- |
| **Watch** | The spread metric ([ADR 0031](../adrs/0031-slot-spread-metric.md)) | Always on — daily samples, dashboards, alerts |
| **Prevent** | [Model-Affine Slot Reservation](../glossary.md#model-affine-slot-reservation) ([ADR 0032](../adrs/0032-model-affine-slot-reservation.md)) | Automatic — no operator action, ever |
| **Cure** | [Model Compaction](../glossary.md#model-compaction) ([ADR 0033](../adrs/0033-operator-initiated-model-compaction.md)) | Operator-initiated, per model, when the decision framework (§4) says so |

## 2. How spread happens

Four mechanics, so you can name the cause when the metric flags a model:

1. **Relocation lifecycle.** Every field retype and every filterability promotion vacates the field's current slot and reserves a new one ([ADR 0016](../adrs/0016-field-type-change-lifecycle.md)). Each relocation is an opportunity for the field to land on a different page than its model-siblings.
2. **Family ceilings.** Each page holds **25 string / 15 int / 10 numeric / 10 datetime** slots. A model with more filterable fields of one family than a page holds is spread *by necessity* — no tool changes that floor; only model redesign does.
3. **Incremental growth.** A model whose fields are registered over months takes whatever page has a free slot at each moment.
4. **Pre-affinity history.** Reservations made before model-affine reservation shipped used a global oldest-free order with no model awareness — models that grew or churned in that era are the likeliest to be fragmented.

There is **no in-place fix**: `ALTER TABLE` on populated pages is forbidden ([ADR 0012](../adrs/0012-immutable-extension-page-ddl.md)), so the only way to move a slot's contents is the tombstone → reserve → backfill → sweep lifecycle. That is exactly what compaction orchestrates — which is why it is a deliberate operation, not a background default.

## 3. Watching the metric

### 3.1 The two events

Both are structured-log records on `source: registry`:

- **`spread_sampled`** — emitted on *every* sample (daily jittered schedule via the Watcher, one-shot after any relocation, or on demand). Feed dashboards and trend analysis from this.
- **`high_spread_model`** — emitted additionally when `pages_occupied >= 2` **and** `excess_pages >= spread_excess_page_threshold`. Alert on this.

Key fields:

| Field | Meaning |
| :--- | :--- |
| `pages_occupied` | Distinct pages holding the model's live filterable slots |
| `theoretical_min_pages` | Fewest pages the model's filterable fields could occupy, given family ceilings |
| `excess_pages` | `pages_occupied − theoretical_min_pages` — **the count of avoidable joins**; `0` is optimal packing |
| `live_slot_count` | Live query-servicing slots (`assigned`/`ready`) behind the sample; `backfilling` and `tombstoned` slots are excluded because they serve no query |
| `trigger` | `periodic`, `post_relocation`, or `on_demand` |

Read `excess_pages`, not `pages_occupied`: a model that genuinely needs three pages (family-ceiling-bound) reports `excess_pages = 0` and needs nothing from you.

### 3.2 Threshold tuning

`spread_excess_page_threshold` defaults to `2` — one avoidable join is deliberately ignored, because a compaction is rarely worth saving a single bounded index scan. Tighten to `1` for latency-critical fleets; raise it if you treat spread as cosmetic. On-demand triage: `bin/stardust spread:report [--tenant=N] [--model=N]`.

### 3.3 Manual measurement, today

The metric is registry-only, so you can measure spread right now with one query (adapted from ADR 0031 §Sampling Method):

```sql
SELECT
  f.model_id,
  COUNT(DISTINCT sa.page_id) AS pages_occupied,
  COUNT(*)                   AS live_slot_count,
  SUM(sa.slot_column LIKE 'i_str\_%') AS str_slots,
  SUM(sa.slot_column LIKE 'i_int\_%') AS int_slots,
  SUM(sa.slot_column LIKE 'i_num\_%') AS num_slots,
  SUM(sa.slot_column LIKE 'i_dt\_%')  AS dt_slots
FROM stardust_slot_assignments sa
JOIN stardust_fields f ON f.id = sa.field_id
JOIN stardust_models m ON m.id = f.model_id
WHERE m.tenant_id = :tenant_id
  AND sa.status IN ('assigned', 'ready')
  AND f.is_filterable = 1
GROUP BY f.model_id;
```

Compute the theoretical minimum by hand from the per-family counts: `max(ceil(str/25), ceil(int/15), ceil(num/10), ceil(dt/10))`. `excess_pages` = `pages_occupied` minus that. The query touches only registry tables — safe to run against production at any time.

## 4. Decision framework: Accept / Compact / Redesign

When `high_spread_model` fires (or your manual query surprises you):

| Situation | Action |
| :--- | :--- |
| `excess_pages` 0–1 | **Accept.** One avoidable join is rarely worth a data migration. |
| `excess_pages >= 2`, model is hot / latency-critical | **Compact** (§6), in a maintenance window. |
| `excess_pages >= 2`, model is low-traffic | **Accept.** The joins are cheap and nobody is waiting on them. |
| `theoretical_min_pages > 1` (family-ceiling-bound) and that is the pain | **Redesign.** Compaction cannot go below the floor — split the model, or demote filterable fields you don't actually filter on. |

Compaction is a real migration (it rewrites the model's filterable projection and each in-flight field temporarily loses filterability), so the bar is "the metric says avoidable, the workload says it matters" — both, not either.

## 5. Prevention hygiene (model-design rules)

Affinity ([ADR 0032](../adrs/0032-model-affine-slot-reservation.md)) is automatic and needs nothing from you. What it *cannot* do, you own at design time:

- **Keep filterable-field counts lean**, and watch the family ceilings (25/15/10/10). A model creeping toward 25 filterable strings is a redesign conversation, not a capacity request — past the ceiling, spread is permanent.
- **Register fields deliberately, up front**, rather than drip-feeding them over months. A model created whole lands compact; a model grown field-by-field gambles repeatedly on free-slot placement (affinity improves the odds; it does not guarantee).
- **Avoid `is_filterable` churn.** Every promotion is a full relocation ([ADR 0009](../adrs/0009-tombstone-based-slot-eviction.md) / [ADR 0016](../adrs/0016-field-type-change-lifecycle.md)) — a re-roll of the field's page placement plus Reconciler and Liberator work. Decide filterability at registration where possible.
- **Remember affinity is forward-only.** It keeps new reservations compact; it never repairs existing spread. Fleets migrating from the pre-affinity era should expect the metric to flag some legacy models once, cure the ones that matter (§6), and then see few new flags.

## 6. Running a compaction

### 6.1 Pre-flight checklist

- [ ] **Dry-run first**: `bin/stardust compact:model --tenant=N --model=N --dry-run`. Read the plan: current spread, chosen target pages, per-field relocations, no-op count, and the **capacity verdict**. No mutation happens.
- [ ] **Liberator healthy?** Compaction double-occupies: each relocated field holds its old (tombstoned) and new (backfilling) slots until the Liberator sweeps the old one. Confirm the Liberator is running and the tombstone backlog is draining, or the capacity check may refuse the plan.
- [ ] **Reconciler workers running?** They execute the actual chunked backfill. A compaction with no Reconciler makes no progress.
- [ ] **Low-traffic window scheduled?** While each field is in flight, **filters on it fail** with a typed exception ([ADR 0004](../adrs/0004-fail-fast-on-unindexed-filters.md)); reads keep working via the JSON payload ([ADR 0013](../adrs/0013-json-payload-as-system-of-record.md)). Warn consumers who filter on the model.
- [ ] **Terminal that survives you walking away** (tmux / screen): the CLI is a long-lived process in sequential mode.

### 6.2 Procedure

```
bin/stardust compact:model --tenant=N --model=N
```

Sequential: the CLI initiates one field's relocation, waits for the Reconciler to backfill and promote it (filterability restored), then moves to the next. **At most one field has filters rejected at any moment.**

> **`--parallel=N` is not implemented.** ADR 0033 describes it as an explicit opt-in that trades filter availability for wall-clock speed; it was deliberately left out of the initial implementation, since it must never be the default and needs multi-checkpoint tracking. The flag is unrecognised today — compaction is always sequential. If you need it, that is a feature request, not a misconfiguration.

The CLI needs a running `bin/stardust reconciler` to make progress: it initiates each relocation and then polls the checkpoint the Reconciler drains. With no Reconciler it fails with a clear message rather than hanging indefinitely.

### 6.3 Monitoring

Watch the registry-source event stream:

`compaction_planned` (once, with `pages_before`, `target_pages`, `fields_to_relocate`, `noop_count`) → per field: `retype_started` → `chunk_complete` (progress) → `promote_to_ready` (filterability restored) → `compaction_complete` (with `pages_before`, `pages_after`, `duration_seconds`).

The built-in success check is the final `spread_sampled` with `trigger: post_relocation` — `excess_pages` should be `0` (or the family-ceiling floor).

### 6.4 Crash / interrupt

Killing the CLI (or losing the box) is benign. In-flight checkpoints are standard retype checkpoints — the Reconciler drains them to completion regardless, restoring that field's filterability. **Resume = re-run the same command**: the planner recomputes the remaining delta, already-relocated fields are no-ops, and the operation converges. There is no cleanup step and no stuck state.

## 7. Troubleshooting

| Symptom | Cause & action |
| :--- | :--- |
| `CompactionCapacityException` at start | Target pages lack free slots (double-occupancy needs headroom, and fragmented deployments are exactly the ones short on it). Check tombstone depth (`SELECT status, COUNT(*) FROM stardust_slot_assignments GROUP BY status`), let the Liberator sweep and/or the Watcher provision, re-run. Nothing was mutated. |
| `RetypeInProgressException` on a field | The field is already mid-retype/promotion. Let the running checkpoint finish (watch for its `promote_to_ready`), then re-run. |
| Nonzero `coercion_null` events during compaction | Pre-existing malformed payload values failing to parse under their own declared type during the identity backfill. This is *visibility, not loss* — the JSON payload stays authoritative ([ADR 0013](../adrs/0013-json-payload-as-system-of-record.md)) and reads fall back to it. Audit the flagged rows; expect nonzero counts on dirty legacy models. |
| `excess_pages` still > 0 after a clean `compaction_complete` | Compare against `theoretical_min_pages` — if they now match, you have hit the family ceiling and this is the floor. The remaining lever is redesign (§4), not another compaction. |
| Compaction makes no progress after initiation | Reconciler workers are down or saturated. The checkpoint is durable; start/scale `bin/stardust reconciler` and progress resumes. |

## 8. References

- [ADR 0031 — Slot Spread Metric](../adrs/0031-slot-spread-metric.md) (the signal; normative event schema and thresholds)
- [ADR 0032 — Model-Affine Slot Reservation](../adrs/0032-model-affine-slot-reservation.md) (the automatic prevention)
- [ADR 0033 — Operator-Initiated Model Compaction](../adrs/0033-operator-initiated-model-compaction.md) (the cure; normative procedure semantics)
- Glossary: [Spread](../glossary.md#spread) · [Model Compaction](../glossary.md#model-compaction) · [Model-Affine Slot Reservation](../glossary.md#model-affine-slot-reservation)
- Lifecycle background: [ADR 0009](../adrs/0009-tombstone-based-slot-eviction.md) (tombstone/sweep), [ADR 0012](../adrs/0012-immutable-extension-page-ddl.md) (why no in-place fix), [ADR 0016](../adrs/0016-field-type-change-lifecycle.md) (the relocation lifecycle compaction rides)
