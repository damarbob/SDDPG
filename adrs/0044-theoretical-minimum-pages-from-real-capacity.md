# 0044 - The Theoretical Minimum Is Derived From Real Page Capacity

**Status:** Proposed
**Created:** 2026-09-05

## Context

ADR `0031` defines the spread advisory's central number:

```text
theoretical_min_pages = max over families f of  ceil( filterable_field_count[f] / capacity[f] )
```

with `capacity = { str: 25, int: 15, num: 10, dt: 10 }`. ADR `0033`'s compaction planner reuses it — deliberately, so that a compaction and the metric verifying it cannot disagree about whether anything was achieved.

The divisor is the **page layout**: how many columns of a family a page is created with. That equalled a page's real capacity only while two things held — every page carried all sixty columns, and all sixty could be claimed. ADR `0034` ended the second (a filterable field's slot must be indexed, so an unindexed column is unclaimable). ADR `0042` sized new pages to `demand + k`, and ADR `0043` made a page's column set *equal* to its claimable set. A page's capacity for a family is now `k` plus whatever demand it was provisioned against, and pages are heterogeneous by design.

ADR `0031` predicted this precisely, in its own Consequences:

> `theoretical_min_pages` assumes the standard per-family page layout. If a future ADR introduces heterogeneous page layouts, the min-pages formula must be revised in lockstep, or the metric will misreport excess.

Three ADRs introduced them and the formula was not revised.

### Measured

MySQL 8.0.13, default `Config::$pageIndexHeadroom = 4`, promoting filterable `str` fields one at a time — register, write, Watcher tick, Reconciler drain:

```text
5 fields  → pages_occupied=2  theoretical_min_pages=1  excess_pages=1
9 fields  → pages_occupied=3  theoretical_min_pages=1  excess_pages=2  → high_spread_model ALERT
```

Every page carries exactly four `str` columns, so both layouts are optimal and the true floors are 2 and 3. `ceil(n/25)` answers 1 for both. **Five fields is the first size that misreports; nine is the first that alerts.**

Two symptoms follow from the one number:

- **The advisory cries wolf.** `excess_pages` is non-zero for a model that cannot be packed tighter, and at nine fields it crosses `Config::$spreadExcessPageThreshold` and fires `high_spread_model` — telling an operator to compact something already compact.
- **Compaction then refuses.** `CompactionPlanner` starts its search at `minPages` and widens; every candidate set smaller than `pagesBefore` fails to fit, because assignment counts *real* per-page free inventory. It exhausts the loop and throws `CompactionCapacityException`, `--dry-run` included.

Nothing is corrupted — assignment reads actual inventory, so a plan can never over-commit a page. This is a reporting and refusal defect. It is also self-correcting once the floor is right: the planner's early return is `pagesBefore <= minPages`, so a correct floor turns both symptoms into one no-op plan and `excess_pages = 0`.

### There is no constant to substitute

The obvious repair — divide by `k` instead of by 25 — is wrong for the same reason the original is. Pages are heterogeneous by construction (ADR `0043` consequence: two page shapes exist permanently, and a demand-sized page is wider in the family it was provisioned for). A page-independent capacity number does not exist, so the floor has to be read off the actual pages.

### The exception's advice is separately wrong

`CompactionCapacityException` tells the operator to *"let the tombstone backlog drain, let the Watcher provision, and re-run."* Draining is sound — a Liberator sweep returns `tombstoned` slots to `free` on pages the model occupies. Provisioning cannot help under any circumstances: `rankCandidates()` draws candidates from `$pagesOccupied` alone, which is ADR `0033`'s documented v1 restriction that compaction consolidates and never migrates a model onto a page it has never touched. A page the Watcher provisions in response to that advice is not a candidate.

## Decision

**`theoretical_min_pages` is computed from the capacity the candidate pages actually offer this model, not from the page layout.**

1. **Candidate pages are the pages the model occupies**, matching ADR `0033`'s v1 restriction. Free capacity on a page the model has never touched is not reachable by any operation, so counting it would report a floor nothing can deliver.

2. **A candidate page's capacity for a family is `indexed free + this model's own live slots there`** — exactly what compaction can use: fields already on a target page stay put, and everything else needs a free indexed slot. This keeps `excess_pages → 0` a true success criterion rather than two subsystems agreeing by luck.

3. **The minimum is the fewest candidate pages that cover the counts, per family, roomiest first:**

   ```text
   m[f]    = fewest pages whose capacity for f, taken largest first, covers count[f]
   minimum = max over families f of m[f]
   ```

   Still a max over families, never a sum — a page serves all four simultaneously. Where every page carries the same number of columns this reduces exactly to `ceil(count[f] / capacity[f])`, so ADR `0031`'s worked example (30 `str` + 5 `int` ⇒ 2) is unchanged.

4. **Taking each family's roomiest pages independently makes the result a lower bound**, not an exact packing: two families may want different pages. That is the safe direction and is what "theoretical minimum" means. On engine-provisioned pages it is also exact, because ADR `0042` gives every page columns in every family.

5. **The floor and the assignment budget stay distinct.** The floor counts `free + own`; assignment spends `free` alone. A slot the model already holds is capacity for staying put, never capacity for taking a move — conflating them would resurrect the double-occupancy trap ADR `0033` names.

6. **The `CompactionCapacityException` message drops its provisioning advice**, keeping the drain-and-re-run half and stating that a newly provisioned page is not a candidate.

Both subsystems read the per-page capacity through one shared component, on the same reasoning that made the formula shared in the first place. The spread advisory pays one additional aggregate query per sampling run — over the whole pool, grouped by page, not per model — and stays registry-only, so it remains safe to run over every model, every day.

## Consequences

**Positive:**

- The advisory stops firing at models that are already optimally packed, which is the whole point. `excess_pages = 0` regains its meaning: *nothing can pack this tighter*.
- `compactModel()` stops refusing those models. They take the existing `pagesBefore <= minPages` early return and report a no-op plan, `--dry-run` included.
- The metric and the operation agree by construction, now on both the formula and its input.
- No DDL, no migration, no new configuration, no new event. The number changes; nothing about its shape does.

**Negative:**

- **The metric is now time-varying.** A model's floor moves when an unrelated model claims or releases slots on a page it shares. Two samples of an unchanged model can differ. This is inherent to measuring against reachable capacity rather than a constant, and it is the price of the metric agreeing with the operation.
- **A model fragmented across pages whose spare slots other models hold now reports `excess_pages = 0`.** It is paying avoidable joins in the abstract, but no operation available to the operator can reclaim them, so the advisory stays quiet. Honest, but it means the metric no longer surfaces every join cost.
- **Spread forced by headroom policy is invisible.** Nine `str` fields at `k = 4` occupy three pages and report zero excess. Those two extra joins are real, and the lever for them is `Config::$pageIndexHeadroom` on future pages, not compaction. `pages_occupied` is on both events and remains the raw signal; deliberately, no second number was added.
- **`CompactionCapacityException` becomes markedly rarer.** With no free capacity anywhere, a page's capacity is exactly what the model already holds there, so covering the counts takes every page and the floor always equals `pages_before` — a starved model is a no-op, never an error. The exception remains reachable where the per-family bound is loose: families that each fit on one page but disagree about which.
- The spread advisory runs two queries per sampling run instead of one.

**Rejected alternatives:**

- **Divide by `k`.** Same class of error as dividing by 25 — a constant where there is no constant. Demand-sized pages are wider than `k` in the family they were provisioned for, and pre-`0043` pages are wider in all four.
- **Count a page's total columns rather than free-plus-own.** A structurally stable number, immune to other models' activity. Rejected because a compaction could then complete cleanly and still leave `excess_pages > 0` when the spare columns belong to someone else, which breaks the success criterion ADR `0033` is verified by and reintroduces a milder version of the same false alarm.
- **Emit a second number for layout-forced spread** (`layout_min_pages`, or similar). Would keep `k`-forced join cost visible. Rejected as re-introducing the cry-wolf under a new name: it is a number describing no page that exists, and the operator's response to it is a provisioning decision that this advisory does not exist to prompt.
- **Fix only the planner and leave the metric alone.** Would stop the refusal and leave the false alarm, and would break the shared-formula property that `src/Compaction/CLAUDE.md` records as load-bearing.

## Related

- ADR `0031` — Slot Spread Metric. Amended: its published `theoretical_min_pages` formula is superseded by the one above. Its `pages_occupied` / `excess_pages` definitions, its sampling triggers, its event fields and its two-bound alert gate are all unchanged. Its own Consequences section predicted this amendment.
- ADR `0033` — Operator-Initiated Model Compaction. Amended: planning step 2 uses the formula above, and the capacity-failure remedy no longer names Watcher provisioning. Its candidate restriction, its double-occupancy rule and its fail-before-mutation guarantee are unchanged.
- ADR `0043` — Extension Pages Provision Only the Columns They Index. The immediate cause: it made a page's column set equal to its claimable set, and pages heterogeneous.
- ADR `0042` — Index Headroom at Page Provisioning. Sets `k`, and is why every engine-provisioned page carries every family — the property that makes the per-family bound exact in practice.
- ADR `0034` — Non-Filterable Fields Are JSON-Only. The first step away from layout-equals-capacity.
- ADR `0016` / `0004` — Why a relocated field's slot must be indexed, and therefore why capacity counts indexed free slots only.
- ADR `0009` — Liberator reclamation. Why a vacated slot is `tombstoned` rather than `free`, and so is not capacity.
