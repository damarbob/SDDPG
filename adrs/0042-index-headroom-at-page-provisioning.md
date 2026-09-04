# 0042 - Index Headroom at Page Provisioning

**Status:** Proposed
**Created:** 2026-09-04

## Context

ADR `0003` decided that index provisioning is schema-driven and that columns beyond current demand are provisioned unindexed. ADR `0035` sized that demand precisely: the Watcher indexes `clamp(waiters − indexedFree, 1, familyCapacity)` columns per demanded family, and nothing else. ADR `0012` froze the result — index composition is fixed at `CREATE TABLE` and never altered afterwards.

Composed with ADR `0034`, those three decisions produce an outcome none of them states.

ADR `0003`'s Decision originally read: *"When a field is not marked filterable, the slot may still exist for typed retrieval, but it is not considered valid for filtering."* Under that premise, provisioning sixty columns and indexing one was coherent — the other fifty-nine were usable typed storage. ADR `0034` withdrew that clause on 2026-07-12 and annotated `0003` in place with *"The core decision of this ADR (schema-driven, registration-time index provisioning via `is_filterable`) stands unchanged."*

**The core decision does not stand unchanged, because its cost model no longer holds.** ADR `0034` deleted the only consumer of an unindexed slot column. Every production reservation path — `RetypeInitiator`, `RetypeBackfillWorkSource`, `UnmappedFieldReserver`, and compaction's page-pinned variant — passes `requireIndexed: true`; the one `requireIndexed: false` default on `SlotReserver::reserve()` is reached from `docker/seed.php` and tests only. Combined with ADR `0004` (a filterable field's slot must be indexed) and ADR `0034` (only filterable fields hold slots), an unindexed slot column **cannot be legally occupied by anything**. What was conservatism about over-indexing became a guarantee that fifty-nine sixtieths of every page is unreachable, and that a page's usable capacity is permanently equal to the number of columns indexed at its birth.

### What this produces, measured

Against MySQL 8.0.13, promoting the three fields of one model (`city` string, `country` string, `population` int) serially — promote, Watcher tick, Reconciler drain, repeat, which is the ordinary operator sequence:

```text
after promoting city         pages=1  slots=60   free=57   live=1
after promoting country      pages=2  slots=120  free=117  live=2
after promoting population   pages=3  slots=180  free=177  live=3   free_ratio=0.9833
  page 1  indexed: i_str_01
  page 2  indexed: i_str_01
  page 3  indexed: i_int_01
  free-but-unindexed (unclaimable by any production path): 177
```

Three pages for three fields, and the engine flags its own output: `high_spread_model` with `pages_occupied: 3`, `theoretical_min_pages: 1`, `excess_pages: 2`.

The state is **not recoverable**, which is the sharp part:

- `compactModel()` raises `CompactionCapacityException` — *"no smaller page set has enough free slots of the required families to absorb the moves"* — because ADR `0033`'s planner counts indexed free slots, and there are none.
- That exception advises letting the Watcher provision and re-running. The Watcher will not: no family is starved (all three fields hold slots) and the global free ratio is `0.9833` against a `0.20` threshold, so it reports `action: no_action, trigger: none` on every tick, indefinitely.
- ADR `0032` model affinity cannot fire. Affinity can only co-locate onto a page holding an indexed free slot of the field's family, and demand-sized provisioning guarantees there is never one. All three reservations logged `affinity: fallback`; the same three fields promoted *concurrently* land on one page and log `co_located` twice.

So three ADRs aimed at page spread — `0031` measures it, `0032` prevents it, `0033` cures it — are defeated or made redundant by the provisioning policy that creates it. Two of them are structurally inert; the third faithfully reports damage nothing can undo.

The page count tracks promotion *concurrency*, not field count: promoting all three before a Watcher tick yields one page indexing `i_int_01, i_str_01, i_str_02`. Serial promotion is the common case and the worst case.

### Why the fix must land before GA

ADR `0012` forbids adding an index to a populated page. Every page created under the current policy is therefore a permanently one-column page, and no later decision can upgrade it. This defect is not retroactively fixable, which is the argument for deciding it while `0.3.0-alpha.1` is unreleased and no deployment holds such pages.

## Decision

At page provisioning, the indexed-column set is sized as `max(demand_f, k)` columns **of every slot family** `f`, clamped to that family's per-page capacity, where `k` is an operator-tunable headroom constant.

Concretely, `ProvisioningPlanner::indexedColumnsFor()` changes in two ways: it iterates all four families rather than only demanded ones, and its per-family floor rises from `1` to `k`. `k` is `Config::$pageIndexHeadroom`, appended after `$modelPurgeLockRetryBudget`, **default 4** — sixteen indexed columns on a new page against the current one to three.

The rule applies to every provisioning trigger. A page provisioned by `low_capacity` with no pending demand previously carried zero indexes and was therefore pure dead weight; it now carries `4k` indexed columns and is genuine capacity.

Everything else in ADR `0003` stands: provisioning remains schema-driven and registration-time, `is_filterable` remains the source of truth for queryability, and indexes are still incorporated into page DDL at creation rather than added opportunistically in response to query behaviour. ADR `0012` is untouched — this is still creation-time-only index composition with no `ALTER TABLE` on any page, populated or empty. ADR `0016` commitment 4 is untouched — neither retype nor promotion executes DDL synchronously; only the Watcher provisions.

What changes is the *width* of the creation-time set, and the justification is that ADR `0034` removed the alternative use of the columns being left out.

### Evidence

MySQL 8.0.13 Community, `innodb_flush_log_at_trx_commit = 1`, doublewrite on. Two workloads, each variant in a fresh database in its own process to remove ordering effects.

**End-to-end, real engine** — `schemaBuilder` → promote → Watcher → Reconciler → `bulkWrite` of 20,000 entries → `search()` through `MysqlNativeDriver`, three interleaved repetitions, 400 query iterations each. The headroom world pre-provisions one page with sixteen indexed columns and then promotes **serially**; all three fields land on that page, confirming the mechanism.

| metric | headroom (1 page, 16 indexes) | status quo (3 pages, 1 index each) |
| :-- | :-- | :-- |
| `bulkWrite` rows/s | 806, 730, 620 → **719** | 658, 377, 465 → **500** |
| filter on 1 field | 1.41 ms | 1.24 ms |
| filter on 2 fields | 2.44 ms | 2.37 ms |
| filter on 3 fields | 1.78 ms | 1.74 ms |

**Headroom is faster on writes in both independent runs of this comparison** — +44% here and +24% in a second run (means 510 against 410 rows/s), pooling to roughly **+35%** over six repetitions. Individual repetitions overlap in the second run, so treat the effect size as approximate and the direction as the finding. Read differences are within run-to-run spread and no read benefit is claimed.

That second run carried a third layout — one page with only the three indexes the fields actually use — which isolates the two effects at equal scale by varying one thing at a time:

| layout | pages | indexes | `bulkWrite` rows/s | vs baseline |
| :-- | :-- | :-- | :-- | :-- |
| 1 page, 3 indexes | 1 | 3 | 680, 701, 563 → **648** | baseline |
| 1 page, 16 indexes | 1 | 16 | 366, 552, 611 → **510** | **−21%** |
| 3 pages, 1 index each | 3 | 3 | 432, 437, 361 → **410** | **−37%** |

Thirteen extra indexes cost 21%; two extra pages cost 37%. Per unit that is roughly 1.6% per index against 18% per page — **one extra page costs about what eleven extra indexes cost**, and that ratio is the quantitative core of this decision. The −21% figure independently reproduces the paired single-page measurement below, which used a different harness (direct `SlotRowUpserter` upserts rather than the engine's `bulkWrite` path).

The write result is counter-intuitive and worth stating plainly: sixteen index maintenances against one row insert cost **less** than three index maintenances across three row inserts. A write into a three-page layout issues three separate `INSERT … ON DUPLICATE KEY UPDATE` statements into three tables, with three clustered-row inserts and three primary-key lookups. The index count is not the dominant term; the page count is.

**Isolated index ladder** — direct `SlotRowUpserter` upserts, 50,000 rows into a single page, so this measures index maintenance alone with page multiplication factored out. At a 256 MB buffer pool (forward and reverse order, means): 1 index 4,302 rows/s · 3 indexes 4,840 · 16 indexes 3,628 · 32 indexes 2,439 · 60 indexes 1,786. A paired interleaved comparison of 1 versus 16 indexes gave ratios of 67%, 83%, 96%, 70% — **k=4 costs about 21% of single-page write throughput**, and that figure is the honest cost for the one case where headroom buys nothing: a model with exactly one filterable field, which occupies one page either way and carries fifteen idle indexes. Break-even is around two filterable fields.

`k=8` (−43%) and indexing all sixty columns (−59%) are rejected on these numbers. ADR `0003`'s instinct about the extreme was right; it overcorrected for the middle.

**Memory is the real variable.** Repeating the ladder at an 8 MB buffer pool collapses every variant — the same sixteen-index page runs at roughly 1,000 rows/s instead of 3,600. Index footprint per 50,000-row page is 7 MB at one index, 13 MB at three, 33 MB at `k=4`, 57 MB at `k=8`, 99 MB at sixty, against 5.5 MB of row data. The operating requirement is therefore that the buffer pool hold the indexed footprint of the active pages; at `k=4` that is roughly 33 MB per 50,000 rows per page, which is unremarkable on a real host and impossible on a toy one. This belongs in the host-sizing runbook ADR `0027` defers.

### Not decided here

`CapacityReporter` counts unindexed slot rows as free capacity, so the Watcher logs `usable_free_slots: 177` on the three-page database above when the true claimable count is zero. Headroom alone does not fix that — a `k=4` page still registers sixty free slots when sixteen are claimable. **ADR `0043` addresses it** by provisioning only the columns it indexes, so that `free` and `claimable` become the same set with no new column and no new predicate. The alternative considered and rejected was persisting indexability as `stardust_slot_assignments.is_indexed`: a denormalised cache of a physical fact, able to drift from `information_schema`, and still requiring maintenance after the page shape made it redundant. The two ADRs are independent — headroom stands on its own if `0043` is rejected — but `0043` assumes `0042`.

It is worth recording *why* it must not be done alone: switching the global ratio to indexed-only without headroom diverges exactly as ADR `0035` proves. With three pages each carrying one taken index the ratio is `0/3`, below threshold, so the Watcher provisions — and the new page, having no demand, receives no indexes under the current rule, so the ratio stays at zero and it provisions every tick forever. Headroom is what makes any honest capacity gauge safe, whichever route `0043` takes, so this ADR is its prerequisite rather than its substitute.

## Consequences

**Positive:**

- A model's filterable fields cluster on one page under ordinary serial promotion. The three-field model above goes from three pages to one, `excess_pages` from 2 to 0.
- ADR `0032` affinity becomes operative rather than structurally inert: there is now an indexed free slot on the affine page for it to prefer.
- ADR `0033` compaction becomes reachable. The unrecoverable `CompactionCapacityException` state above cannot form, because a smaller page set retains indexed free slots to absorb moves.
- Bulk ingest into a multi-field model gets faster — by 24% to 44% across two runs of the measured three-field case — because page multiplication costs more than index width.
- `low_capacity` provisioning stops producing pages that are pure dead weight.
- Slot inventory stops advertising capacity that no code path can claim — sixteen of sixty columns become genuinely reservable instead of one.

**Negative:**

- A model with exactly one filterable field pays roughly 21% of single-page write throughput for fifteen indexes it never uses. This is the honest cost of the decision and it is real; it is accepted because such models are the minority and because the alternative penalises the majority worse.
- Index storage per page rises from ~7 MB to ~33 MB per 50,000 rows. Pages are provisioned on demand and this is not per-entry overhead, but a deployment with many pages will notice.
- The engine acquires a tuning knob whose wrong setting is expensive in both directions, and whose correct setting depends on buffer pool sizing that the project does not yet document.
- `k` is fixed per page at creation, so changing the default only affects pages provisioned afterwards. Existing pages keep whatever they were born with — this ADR does not escape ADR `0012`, it only chooses a better birth.

**Neutral, but easy to misread:**

- Idle indexed columns do **not** generate `low_cardinality_index` noise. `CardinalitySampler` scans `status IN ('assigned','ready')` only, so a free indexed column is never sampled.
- This does not reopen "typed retrieval" for non-filterable fields. ADR `0034` stands in full; headroom columns are reserved for *filterable* fields that arrive later, not for JSON-only ones.

**Rejected alternatives:**

- **Index all sixty columns.** −59% write throughput and 99 MB of index per 50,000-row page, for capacity most models never claim. This is the outcome ADR `0003` was right to refuse.
- **`k = 8`.** −43% write throughput and 57 MB per page, to buy headroom beyond what field counts justify. The measured knee is between 16 and 32 indexes.
- **`ALTER TABLE … ADD INDEX` on an empty page.** ADR `0012`'s own title ("Empty-Table-Only Schema Changes") permits it and `EmptyTableGuard` exists unused for exactly this. But a page stops being empty at the first write after its first reservation, so it buys one promotion's grace and does not address serial promotion at all.
- **Derive the index set from registered-but-not-yet-filterable fields.** Uses the registry's own knowledge instead of a constant, and would size headroom exactly. Rejected as speculative in a different direction — registration is not a commitment to promote, and a model declaring forty fields would provision indexes for all of them.
- **Leave it and rely on compaction.** Compaction cannot run from the state the policy creates, as measured above. The cure is unreachable from the disease.

## Related

- ADR `0003` — Schema-Driven Index Provisioning. This ADR supersedes its "columns beyond that demand are provisioned unindexed" sizing, narrowly; its schema-driven, registration-time core is unchanged.
- ADR `0034` — Non-Filterable Fields Are JSON-Only. The decision that removed the unindexed column's only consumer, and therefore the premise this ADR re-derives.
- ADR `0035` — Usable Capacity Is a Satisfiability Test. Supplies the index-set sizing rule being widened, and the divergence proofs cited in "Not decided here".
- ADR `0012` — Immutable Extension Page DDL. Unchanged, and the reason this must be decided before GA.
- ADR `0031` / `0032` / `0033` — Spread metric, model affinity, operator-initiated compaction. The subsystems this ADR restores to working order.
- ADR `0004` — Fail-Fast on Unindexed Filters. The rule that makes filterable imply indexed, and thus makes an unindexed slot unoccupiable.
- ADR `0016` — Field Type Change Lifecycle. Commitment 4 (no eager DDL at retype or promotion) is unchanged.
- ADR `0019` — Index Cardinality Policy. Unaffected: free slots are not sampled.
- ADR `0027` — Persistent Process Daemon Execution Model. Host-sizing guidance deferred there now has a concrete buffer-pool requirement to carry.
