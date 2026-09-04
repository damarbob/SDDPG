# 0043 - Extension Pages Provision Only the Columns They Index

**Status:** Proposed
**Created:** 2026-09-04

## Context

Every extension page is created with the full sixty-column slot layout — 25 `i_str_NN`, 15 `i_int_NN`, 10 `i_num_NN`, 10 `i_dt_NN` — regardless of how many of those columns receive an index. ADR `0012` states the consequence plainly: *"columns beyond that demand are provisioned unindexed."*

That was coherent under ADR `0003`'s original wording, where a non-filterable field could hold a slot *"for typed retrieval"*. The spare columns were typed storage waiting for a tenant. ADR `0034` withdrew that clause, and ADR `0042` re-derives what the withdrawal implies for the *index* count. This ADR takes the other half: what it implies for the *column* count.

An unindexed slot column on a new page can no longer be occupied by anything. A filterable field's slot must be indexed (ADR `0004`); a non-filterable field holds no slot at all (ADR `0034`); and an index cannot be added afterwards (ADR `0012`). Sixty columns are created; `4k` of them can ever be claimed.

### The cost is not the bytes

Measured on MySQL 8.0.13 at 400,000 rows, identical data, sixteen identical indexes, varying only the column count:

| | nullable columns | `DATA_LENGTH` | per row |
| :-- | :-- | :-- | :-- |
| exact | 16 | 35.58 MB | 93.27 B |
| full | 60 | 38.59 MB | 101.17 B |

**~8 bytes per row, +8.5%** — which matches the mechanism: InnoDB's NULL bitmap is `ceil(columns/8)` bytes, so 8 bytes against 2, plus rounding. An earlier measurement at 50,000 rows suggested +22%; that was extent-allocation granularity, not row cost, and is superseded by the figure above. A parallel write-throughput comparison across three interleaved repetitions returned 101%, 55% and 85% — inconclusive, and no throughput claim is made here.

So the storage argument is real but minor, and it is **not** the reason for this decision.

### The cost is that the registry lies

`PageProvisioner::insertSlotInventory()` writes one `stardust_slot_assignments` row per column, sixty per page, every one `status = 'free'`. Forty-four of them describe capacity that no reservation path can claim, because every production reservation passes `requireIndexed: true`.

`CapacityReporter` counts those rows. On the three-page database in ADR `0042` the Watcher logs:

```json
{"event":"poll_started","free_ratio":0.9833,"total_slots":180,"free_slots":177,
 "usable_free_slots":177,"pending_waiters":0,"starved_families":[]}
```

**177 usable free slots, of which zero are claimable.** The gauge an operator reads to answer "do I have capacity?" is wrong by the entire inventory, and it stays wrong after ADR `0042` ships: a `k=4` page still registers sixty free slots when sixteen can be taken.

This is the defect. The bytes are a footnote.

### What the spare columns would buy if the policy changed

The strongest case for keeping them is optionality — a future decision might allow filtering or sorting on a typed unindexed column. That option was priced rather than assumed. At 400,000 rows against the engine's real join shape:

| operation | typed unindexed column | JSON payload |
| :-- | :-- | :-- |
| string, after an indexed predicate | 0.50 ms | 0.53 ms |
| string, no index (scan) | 16.72 ms | **14.71 ms** |
| number range, after an indexed predicate | 0.73 ms | 0.94 ms |
| number range, no index (scan) | 159 ms | 179 ms |

The indexed predicate alone is 0.54 ms.

A typed column is worth **nothing for strings** — JSON is marginally faster on a scan, because the payload sits in `entry_data`, which the query already reads for assembly, while the typed column costs an extra page-table read. It is worth **10–22% for numbers**, where JSON pays a parse and cast per row. Against the ~218× that an index is worth on the same query, that margin is not an architectural option; it is a rounding error on a query ADR `0004` refuses to compile.

The foreclosed option is therefore: making a forbidden operation about fifteen percent less catastrophic.

## Decision

An extension page is created with exactly the slot columns it indexes. `PageProvisioner::buildPageDdl()` and `insertSlotInventory()` take the same column list — the one ADR `0042`'s planner produced — instead of `allSlotColumns()`.

Consequences that follow directly:

1. **`free` and `claimable` become the same set.** Every inventory row describes a column that a reservation can take, so `CapacityReporter`'s counts and the Watcher's `free_ratio` become true statements without a new column, a new predicate, or a schema migration.
2. **`SLOTS_PER_PAGE = 60` stops being a fact** and is removed. A page's capacity is `4k`, readable from its own inventory. The 25/15/10/10 layout survives as the per-family upper bound on `k`.
3. **`IndexedSlotPredicate` stays.** Pages provisioned before this decision keep sixty columns and their unindexed inventory rows, so the reserver and the capacity reporter must continue to distinguish. This is forward-only, per ADR `0012`; no existing page is altered and no inventory row is deleted.
4. **`k` may change between provisionings.** Pages are already heterogeneous in index composition — the ADR `0042` repro has `i_str_01` on page 1 and `i_int_01` on page 3 — and every consumer reads per-page inventory rather than assuming a uniform layout. Raising `k` does not widen existing pages; lowering it does not narrow them.

`allSlotColumns()` remains as the validation vocabulary — the set of legal slot column names — which is its only other use.

## Consequences

**Positive:**

- The capacity gauge stops lying, which is the whole point. `usable_free_slots` becomes a number an operator can act on.
- Usable slots per page rises from 1–3 (today, demand-sized) to `4k`, so table proliferation improves markedly: roughly 190 page tables for a thousand three-field tenants instead of one to three thousand.
- ~8 bytes per row saved, and one knob (`k`) now governs page capacity instead of two disconnected numbers.
- The `stardust_slot_assignments.is_indexed` column is not needed. Nothing is denormalised, nothing can drift.

**Negative:**

- **Deployments end up with two page shapes.** Sixty-column pages provisioned before this lands keep their forty-four phantom inventory rows forever, so `IndexedSlotPredicate` and the two-shape reasoning are permanent, not transitional. At `0.3.0-alpha.1` with no production deployments the population of such pages is zero, which is the argument for deciding now.
- A page can no longer be widened by a hypothetical future `ALTER` on an empty page, because the columns are not there to index. ADR `0012` already made that path near-worthless — a page stops being empty at the first write after its first reservation — but this closes it.
- The typed-unindexed-column option is given up permanently for new pages, worth 0–22% on operations the engine forbids.
- Two smoke tests reference `SLOTS_PER_PAGE` and must be rewritten against per-page inventory.

**Rejected alternatives:**

- **`stardust_slot_assignments.is_indexed`.** Fixes the gauge by annotating the lie rather than removing it: a denormalised copy of an `information_schema` fact, able to drift, requiring a `Bootstrapper` probe, and still there after the page shape made it redundant.
- **Keep sixty columns, accept the phantom inventory.** Leaves the operator gauge wrong forever and leaves `usable_free_slots` reporting numbers that are off by the entire unindexed population.
- **Suppress inventory rows for unindexed columns but still create the columns.** Fixes the gauge and keeps the `ALTER`-an-empty-page option. Rejected because it introduces a third category — a column that physically exists but the registry does not know about — and the Liberator, the reserver and the spread metric would each have to agree on which set they mean.

## Related

- ADR `0042` — Index Headroom at Page Provisioning. Assumed by this ADR: without headroom, "provision only what you index" would mean one-column pages. `0042` stands alone if this one is rejected.
- ADR `0034` — Non-Filterable Fields Are JSON-Only. Removed the only consumer of an unindexed column; this ADR removes the column.
- ADR `0003` — Schema-Driven Index Provisioning. Its withdrawn "typed retrieval" clause is what the spare columns existed for.
- ADR `0012` — Immutable Extension Page DDL. Unchanged; this ADR is forward-only under it and closes the empty-page `ALTER` option it nominally allowed.
- ADR `0004` — Fail-Fast on Unindexed Filters. The rule that makes an unindexed slot unoccupiable, and that refuses the queries the priced option would have served.
- ADR `0035` — Usable Capacity Is a Satisfiability Test. Its `usable_free_ratio` gauge becomes meaningful once `free` implies `claimable`.
- ADR `0031` / `0032` / `0033` — Spread metric, affinity, compaction. All read the same inventory, and all benefit from it being true.
