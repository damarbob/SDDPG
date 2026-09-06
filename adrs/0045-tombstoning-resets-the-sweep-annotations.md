# 0045 - Tombstoning Resets a Slot's Sweep Annotations

**Status:** Proposed
**Created:** 2026-09-06

## Context

ADR `0009` specifies the slot eviction lifecycle — **sever → tombstone → sweep → reclaim** — and the Liberator's chunked nullification with a `sweep_cursor_id` checkpoint. It says nothing about what that checkpoint means when a slot goes round the lifecycle a *second* time, because at the time it was written the reuse case was theoretical.

`stardust_slot_assignments` carries two per-sweep annotations:

- `sweep_cursor_id BIGINT NULL` — the highest `entry_id` the current sweep has nullified. `SlotSweeper::sweep()` opens with `$cursor = $slot->sweepCursorId ?? 0`.
- `sweep_gap_count INT NOT NULL DEFAULT 0` — incremented on ADR `0009`'s gap path, when a chunk is abandoned after `deadlockRetryBudget` consecutive lock failures.

**Nothing resets either one.** Grepping every write in `src/` returns exactly two sites, both inside the sweeper:

```text
SlotSweeper:215   UPDATE … SET sweep_cursor_id = ?                        (advance)
SlotSweeper:253   UPDATE … SET sweep_cursor_id = ?, sweep_gap_count = …   (gap path)
```

`LiveSlotTombstoner::tombstone()` writes `field_id`, `status`, `tombstoned_at` and `updated_at`. The reclaim at `SlotSweeper:228` writes `status` and `field_id`. `SlotReserver::reserveCore()` never touches either column. So both values survive `tombstoned → free → assigned → tombstoned` intact, and a column's second sweep resumes from where its *first occupant's* sweep finished.

### The consequence is exposure, not lost work

A sweep that starts too late does not stall — it completes. Every page row at or below the stale cursor keeps the previous field's values, `sweep_complete` fires, the slot returns to `free`, and the next reservation hands a filter a column containing another field's data.

**Measured on MySQL 8.0.13**, driven entirely through the facade — a model with 20 entries and one indexed `i_str_01` column, field `alpha` demoted and swept, field `beta` promoted onto the recycled slot and backfilled, then `beta` demoted and swept:

```text
AFTER TOMBSTONE 1: cursor=NULL  gap=0
AFTER SWEEP 1:     status=free         cursor=20  non_null=0
AFTER RESERVE 2:   status=backfilling  cursor=20  field_id=<beta>
AFTER BACKFILL:    non_null=20
AFTER TOMBSTONE 2: cursor=20
SWEEP 2 EVENTS:    sweep_started, sweep_chunk(rows_nullified=0, sweep_cursor_id=20), sweep_complete
AFTER SWEEP 2:     status=free         cursor=20  non_null=20
```

The reclaimed slot holds twenty rows of `beta`'s data. **There is nothing in the event stream**: `sweep_chunk` reports zero rows nullified at cursor 20 and `sweep_complete` follows, and both are honest for the range actually walked.

Three things make this worse than a first reading suggests.

- It is reachable from **plain facade calls with no compaction involved** — `promoteFieldToFilterable` → `demoteFieldFromFilterable`, twice. Every **retype relocation** also tombstones and re-reserves, so ADR `0033` compaction walks a model through this state field by field, by design.
- **A recycled slot is the *preferred* reservation candidate, not a leftover.** `SlotReserver`'s candidate query orders `a.page_id, a.id`, so a reclaimed column — a low inventory id on an old page — outranks every ADR `0042` headroom column on a newer page and every higher-id sibling on its own.
- Whether the residue is then *visible* depends on who reserves next. A retype or promotion backfill rewrites the whole `(tenant, model)` partition and papers over it for that model; the ADR `0007` exhaustion path does not — `UnmappedFieldReserver` reserves a slot for a field whose backfill covers only the queued entries, so rows written before the queue existed keep the residue and match filters they should not.

This is precisely the "data bleeding" failure ADR `0009` exists to prevent, arriving through the reuse of a slot the same ADR declared safe to reuse.

### The omission is an oversight, not a decision

`SlotSweeper:222-226` carries a deliberate comment saying `sweep_gap_count` is preserved across the reclaim on purpose, and that **"the SlotReserver is the right place to reset it on the next `free → assigned` transition"** — a reset the reserver does not perform, for either column. The reasoning about what a recycled slot should carry forward was written down and then not implemented on either side.

## Decision

**The UPDATE that flips a slot to `tombstoned` also clears both sweep annotations, in the same statement and therefore in the caller's transaction:**

```sql
UPDATE stardust_slot_assignments
   SET status = 'tombstoned', tombstoned_at = ?, updated_at = ?,
       sweep_cursor_id = NULL, sweep_gap_count = 0
 WHERE id = ?
```

1. **A tombstone is the start of a sweep, and a sweep starts at the beginning of the page.** `sweep_cursor_id` describes the sweep in progress and nothing else. `NULL` and `0` are the same state a freshly provisioned inventory row is in, so a recycled slot and a new one are indistinguishable to the sweeper.

2. **This binds both tombstone sites.** `Slot\LiveSlotTombstoner::tombstone()` is the shared two-step used by `RetypeInitiator` and `DeleteFieldInitiator`; `Delete\ModelPurgeWorkSource` inlines the same two-step for a whole model's slots at once. A reset in only one of them recycles stale cursors through the other.

3. **Both sites keep their `status IN ('assigned','backfilling','ready')` guard**, which is what makes the reset safe: a slot that is *already* `tombstoned` — one the Liberator may be sweeping right now — matches neither statement and keeps its cursor. No path resets an in-flight sweep.

4. **The reclaim continues to preserve both.** `tombstoned → free` still writes only `status` and `field_id`. The annotations therefore survive `free` and `assigned` and die at the start of the next sweep, which is the post-mortem semantic `SlotSweeper`'s comment wanted: an operator inspecting a reclaimed or re-reserved slot still sees the gap count from the sweep that produced it.

5. **`SlotReserver` is not the right place, and its comment is withdrawn.** Resetting at `free → assigned` was the alternative on the table. It is rejected on three grounds:
   - **It fails open.** A sweep that starts too early costs one wasted pass of an idempotent `SET col = NULL`; one that starts too late loses data. Tombstone time is the failing-closed end of the window.
   - **A slot reclaimed and never re-reserved would keep a stale cursor indefinitely**, in a registry column an operator reads.
   - It would blank the gap count *before* anyone could act on it, which is the opposite of the reason the comment gave for preserving it.

6. **`sweep_gap_count` is reset alongside the cursor, not separately.** A sweep that has not started has had no gaps, and the counter is only interpretable relative to one sweep of one occupant. Resetting the cursor without it would leave a slot permanently flagged for an occupant that no longer exists.

No DDL: `sweep_cursor_id` is already `NULL DEFAULT NULL` and `sweep_gap_count` already `NOT NULL DEFAULT 0`. No new configuration, no new event, no change to any event's payload.

## Consequences

**Positive:**

- Closes the exposure **for a sweep that ran to completion without taking the gap path** — which is every sweep under normal operation. A slot reclaimed from an uninterrupted sweep is empty, which is the property ADR `0009`'s reclaim step claims and what makes reuse safe.
- **It does not close the gap path, and that is not a shortfall of this decision but a separate accepted risk in ADR `0009`.** After `deadlockRetryBudget` consecutive lock failures the sweep abandons a chunk, advances past it, and still reaches `sweep_complete` — so the slot reaches `free` with the abandoned range still populated. Measured on 8.0.13 (twenty rows, `chunkSize = 10`, one gap): the slot reclaims to `free` holding **ten** of the previous occupant's values, and `SlotReserver` then hands that same slot to the next field, which takes it `assigned` with the ten rows still there. `LiberatorDeadlockRetryTest::testThreeConsecutiveDeadlocksTriggersSweepGap` has asserted the residue half of that since Phase 6a. The blueprint justifies accepting a gap on the grounds that "the field's data is still authoritative in `entry_data.fields`" — which answers whether the *departing* field loses data, not whether the *arriving* one inherits it. That second question is unaddressed by ADR `0009` and is out of scope here.
- **What this decision does contribute to the gap path is attribution.** `sweep_gap_count` survives the reclaim *and* the next reservation (measured above: `gap = 1` on the freshly `assigned` slot), and now resets at each tombstone rather than accumulating across occupants. So a non-zero counter on a live slot reads precisely as **"this column's current contents may include residue from the sweep that preceded them"**, where before ADR 0045 it meant only "some occupant, at some point, had a gap" and could not be attributed to anything. The signal was already there; it was not previously scoped to one sweep.
- The guarantee is now enforced at the point the lifecycle starts rather than depending on which of several reservers happens to take the slot next, and it holds for the ADR `0007` exhaustion path, which has no backfill wide enough to paper over residue.
- The invariant is one clause on a statement that already exists in both tombstone sites; there is no new caller, no new collaborator and no ordering requirement beyond the transaction the caller already owns.
- ADR `0009`'s gap remedy becomes reachable. The Liberator blueprint tells an operator reviewing a gap that they may "re-tombstone the slot to retry sweep over the gap range" — which for a slot severed through the facade now genuinely re-walks the range, where before it resumed past the gap and nullified nothing.

**Negative:**

- **A re-tombstoned slot re-walks rows it already nullified.** Nullification is idempotent (`SET col = NULL`), so the cost is one extra pass of chunked UPDATEs over the page, paid once per lifecycle rather than per occupant. This is the deliberate price of failing closed.
- **The gap count is no longer a lifetime total for the slot.** An operator watching one physical column across many occupants sees the counter zeroed at each tombstone. This is the intended reading — the number describes a sweep, not a column — but it does mean the registry row is not a place to accumulate long-run contention history for a page.
- **A slot already `tombstoned` when a model purge sweeps its model is not reset**, by the same guard that protects an in-flight sweep. Such a slot was severed by an earlier operation and is mid-sweep with a cursor that is correct for it; the purge has no reason to restart it. Correct, but it means "every slot of a deleted model starts its sweep from zero" is not quite true, and the exception is worth knowing when reading a purge's sweep timings.

**Rejected alternatives:**

- **Reset in `SlotReserver` on `free → assigned`**, as `SlotSweeper`'s comment proposed. Rejected per Decision 5: fails open, leaves stale cursors on never-reused slots, and discards the gap count too early.
- **Reset in the reclaim step** (`tombstoned → free`), which is also a single-site change and also fails closed. Rejected because it destroys the annotations at exactly the moment an operator would want to read them — the sweep has just finished, and `sweep_gap_count` is at its most informative.
- **Have `SlotSweeper::sweep()` ignore the stored cursor and always start at zero**, deriving safety from idempotence alone. Rejected because it deletes ADR `0009`'s crash-resumption property, which the Liberator blueprint requires and `LiberatorResumeTest` verifies; a restart mid-sweep of a large page would restart from the beginning every time.
- **Have the Liberator detect reuse** (e.g. compare `tombstoned_at` against the cursor's provenance). Rejected as inventing state to recover information the tombstone already has; the tombstone is the event that means "this cursor is stale".

## Related

- ADR `0009` — Tombstone-Based Slot Eviction over Immediate Reuse. **Amended.** Its sever/tombstone/sweep/reclaim lifecycle, its chunked nullification, its deadlock/gap path and its crash-resumption property are all unchanged; this ADR adds the annotation lifecycle across *repeated* passes through it, which `0009` does not specify. The "data bleeding" hazard in `0009`'s Context is the hazard this closes.
- ADR `0009`'s **gap policy specifically.** Unchanged and untouched by this ADR, but it is the remaining route to the same residue, and its acceptance rationale answers a different question than the one reuse raises. Worth revisiting as its own decision.
- ADR `0029` — The Liberator's sweep omits the tenant predicate. Unchanged, and load-bearing here: it is why an orphaned tombstone is sweepable at all, and why the sweep is keyed on `(page, slot_column)` for `id > sweep_cursor_id` — which is what makes the cursor the whole safety boundary.
- ADR `0017` — Schema Registry as Coordination Contract. The reset rides in the caller's transaction alongside the registry mutation and the schema-version bump, as every other status transition does.
- ADR `0007` — Write-availability fallback. `UnmappedFieldReserver`'s narrow backfill is why the exposure is not self-healing.
- ADR `0034` — Non-Filterable Fields Are JSON-Only. Demotion now always tombstones a live slot, which is what makes the promote/demote cycle a routine facade operation rather than a legacy shape.
- ADR `0037` / `0038` — Field and model deletion. `0037` shares `LiveSlotTombstoner`; `0038` is the second tombstone site this ADR binds.
- ADR `0033` — Operator-Initiated Model Compaction. Walks a model through tombstone-and-recycle field by field, so it is the highest-volume producer of the state this fixes.
- ADR `0042` / `0043` — Index headroom and indexed-only page provisioning. They did not cause this defect, but they changed how a recycled column is re-reserved: a reclaimed low-id slot outranks headroom columns on newer pages in `SlotReserver`'s ordering.
