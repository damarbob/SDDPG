# 0039 - Compaction Refuses to Plan Mid-Lifecycle

**Status:** Proposed
**Created:** 2026-08-27

## Context

[`0033`](0033-operator-initiated-model-compaction.md) relocates a model's live filterable slots onto a minimal page set, and names the final [`0031`](0031-slot-spread-metric.md) spread sample as "the operation's built-in success confirmation: `excess_pages` at (or near, if family ceilings force >1 page) zero". The planner and the metric therefore read the **same population** — `status IN ('assigned','ready')` joined to `is_filterable = 1` — deliberately, so that the operation and the signal verifying it cannot drift.

A field mid-lifecycle is in **neither** status. [`0016`](0016-field-type-change-lifecycle.md)'s initiation tuple tombstones the old slot and reserves the replacement as `backfilling`, so for the whole drain window the field holds one `tombstoned` slot and one `backfilling` slot and is invisible to both. For the metric that is correct and intended: a `backfilling` slot services no query, so it contributes no join cost. For the *planner* it is a hole — the field is going to land somewhere, and the plan is computed as though it were not.

The consequence is not corruption. It is that both ends of the report are wrong. Measured on MySQL 8.0.13 on 2026-08-27, while probing the retype re-runnability defect:

```text
true starting occupancy:  3 pages, excess_pages = 2
beta stranded relocating -> page 3 (checkpoint 'running')
planner sees only alpha(p1) and gamma(p3)

compact() REPORTED:   pagesBefore=2  pagesAfter=1  relocations=1
reality once beta drains normally:   alpha p1, gamma p1, beta p3
ADR 0031 spread:      pagesOccupied=2  excess_pages=1        MISMATCH
```

`compaction_complete` fires carrying `pages_after: 1`, and `spread:report` continues to show `excess_pages: 1`. The design's success signal contradicts the operation claiming to have delivered it.

Three points bound the problem:

- **The stranded field is not stranded.** A `running` checkpoint is claimable independently of the planner, so the Reconciler drains it normally. An earlier framing of this finding claimed the field was left unfilterable; that was wrong.
- **Capacity accounting is unaffected.** Free capacity counts `status = 'free'`, so a `backfilling` slot is never double-booked and no plan over-commits a page.
- **The window is not compaction-specific.** An ordinary `retypeField()`, `promoteFieldToFilterable()` or `demoteFieldFromFilterable()` opens the same window on the same model, so the hole is reachable without any compaction having run.

The reachable states are "the planner's population is complete" and "the planner's population is knowably incomplete". Only the second needs a decision.

## Decision

**Compaction refuses to plan while any field of the target model has a running `retype_field_{id}` checkpoint.** `CompactionService::plan()` raises `RetypeInProgressException` naming the model and the remedy: the checkpoint is durable, a running Reconciler will drain it, wait and re-run.

Four things this pins.

**The planner's population does not change.** It stays [`0031`](0031-slot-spread-metric.md)'s, which is the invariant this ADR exists to protect rather than to trade away. Compaction either reports numbers the spread metric agrees with, or declines to report.

**The guard is on `plan()`, not `compact()`, so `--dry-run` is refused too.** This is the same placement [`0038`](0038-model-deletion-lifecycle.md) chose for the deleting-model guard, and for the same reason: a dry run's entire purpose is to report numbers, so it is the surface where a wrong number does the most damage. Observation during the window is not lost — `spread:report` is registry-only, unaffected by this ADR, and already correct.

**The guard is checkpoint-keyed, not compaction-keyed.** Any running retype checkpoint on any field of the model trips it, including one opened by an ordinary promotion. Narrowing it to relocations would leave the identical defect reachable from `promoteFieldToFilterable()`.

**It is a guard, not a lock.** Two operators can still both pass it in the window between one's `plan()` and its first `initiateRelocation()`; [`0016`](0016-field-type-change-lifecycle.md)'s per-field `RetypeInProgressException` catches the actual collision. Closing that window needs a lock and is deliberately out of scope — the failure it would prevent is already loud.

`RetypeInProgressException` is reused rather than split. The taxonomy splits when two conditions *mean* different things; here the caller-facing meaning and the remedy are identical ("a retype lifecycle is in flight on something you targeted; wait for the Reconciler"), and a `CompactionInProgressException` would additionally misname the ordinary-promotion case.

### This narrows ADR 0033's "Resume is re-run"

[`0033`](0033-operator-initiated-model-compaction.md) states that if the CLI crashes nothing is stuck, and that "re-running `compact:model` recomputes the remaining delta". Convergence and no-stuck-state both survive unchanged. What does not survive is the **immediacy**: a re-run issued inside the drain window is now refused rather than served.

That is a real narrowing and is recorded as such. It is worth noting that the affordance being narrowed did not work as written — a re-run inside the window did not recompute the remaining delta, it computed a delta that omitted the in-flight field. The choice is between refusing the re-run and answering it wrongly.

### Rejected alternatives

**Widen the planner's population to include `backfilling`, pinning those fields as immovable no-ops and forcing their destination pages into the target set.** This produces a correct plan during the window instead of refusing, and was the most tempting option. Rejected because `pages_before` would then be computed over a population `SpreadSampler` does not measure, so the two would report different numbers for the same model — dismantling the parity that makes `excess_pages → 0` a success criterion, in an ADR whose purpose is to defend it. It also pins a destination page the *reserver* chose, not the planner: a field mid-promotion landed wherever [`0032`](0032-model-affine-slot-reservation.md) affinity put it, and forcing that page into the target set can make the resulting plan strictly worse than waiting.

**Leave planning alone and report honestly, counting in-flight fields separately in the plan and the events.** The smallest change, and it does fix the lying numbers. Rejected because it leaves the operation failing its own acceptance criterion and merely announcing that it did: the operator is told `excess_pages` will not reach zero, with no recourse offered but to wait and re-run — which is what the refusal says, earlier and without a data migration in between.

**Do nothing and document the window.** Rejected: the two signals contradicting each other is precisely the failure mode [`0031`](0031-slot-spread-metric.md) and [`0033`](0033-operator-initiated-model-compaction.md) took pains to prevent, and an advisory metric that disagrees with the tool acting on it trains operators to distrust both.

## Consequences

**Positive.**

- Compaction's reported `pages_before` / `pages_after` and the [`0031`](0031-slot-spread-metric.md) spread sample can no longer disagree. `excess_pages → 0` is a criterion again.
- `--dry-run` never prints a number it cannot stand behind.
- One operator cannot compact a model another operator is still compacting — a side effect of the checkpoint keying, and the only shape considered here that delivers it.
- No new schema, no new event name, no new exception class, no new collaborator, and no constructor change: `CompactionService` already holds the `RetypeCheckpointRepository` the guard reads through.

**Negative.**

- An operator whose compaction died mid-relocation cannot re-run immediately; they must let the Reconciler drain first. The refusal is transient and self-clearing, and its message says so, but it is a new way for `compact:model` to decline.
- Compaction of a large model now has a new dependency on the Reconciler being healthy that is visible *before* any work starts rather than after. This is arguably clearer, but it moves the failure earlier.
- A model with a permanently stuck checkpoint — one manually left `running` with no Reconciler ever draining it — cannot be compacted until an operator resolves the checkpoint. There is no override flag, deliberately: the escape hatch would reintroduce exactly the wrong-numbers path this ADR closes.
- The guard reads one extra indexed row per `plan()` call. Immaterial: compaction is operator-initiated and long-running, never on a request path.

**Verification.** The `LIKE` pattern is escaped through `Support\LikePattern`, and this was probed against real MySQL 8.0.13 rather than assumed: an operator-supplied Backfill Pump job named `retypeXfieldY99` is **matched** by the unescaped `retype_field_%` and **not matched** by the escaped form, with the genuine `retype_field_99` matched by both. A stray match here costs a spurious refusal rather than a spurious write, but the same wildcard hazard is unrecoverable in [`0038`](0038-model-deletion-lifecycle.md)'s namespace, so the escaping is uniform across all four.

## References

- [`0033`](0033-operator-initiated-model-compaction.md) — Operator-Initiated Model Compaction (narrowed here: "Resume is re-run")
- [`0031`](0031-slot-spread-metric.md) — Slot Spread Metric (the population and the success criterion)
- [`0016`](0016-field-type-change-lifecycle.md) — Field Type Change Lifecycle (the window, and the per-field guard)
- [`0038`](0038-model-deletion-lifecycle.md) — Model Deletion Lifecycle (the `plan()`-placement precedent)
