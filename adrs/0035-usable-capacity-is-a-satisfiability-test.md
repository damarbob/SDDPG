# 0035 - Usable Capacity Is a Satisfiability Test, Not a Ratio

**Status:** Proposed
**Created:** 2026-08-10

## Context

ADR `0016`'s 2026-07-13 clarification and [`blueprints/watcher_reconciler_daemons.md`](../blueprints/watcher_reconciler_daemons.md) AC#1 pin the Watcher's provisioning trigger to **usable** capacity rather than raw capacity:

> "Low capacity" means low **usable** capacity: free slots that cannot satisfy any pending demand shape (e.g. unindexed free slots when every waiting field requires an index) do not count toward the provisioning threshold.

The intent is unambiguous and correct — a page full of unindexed free columns must never mask a real shortage of indexed capacity. ADR `0034` sharpened the stakes: now that non-filterable fields are JSON-only, every reservation path with a production caller demands an *indexed* slot, so unindexed free inventory is capacity nobody can claim.

The problem surfaced during implementation. The clause specifies a **numerator** — free slots that can satisfy pending demand — and no **denominator**. Both candidate denominators produce a threshold comparison that never stabilises:

- **Global denominator** (`usableFree / totalSlots`). One `dt` waiter across ten pages gives a ratio near `0.0017`. Provisioning a page adds one usable slot to the numerator but sixty slots to the denominator, so the ratio *falls further*. The daemon provisions on every tick, forever.
- **Usable denominator** (`usableFree / usableTotal`, both restricted to indexed slots of demanded families). This converges from an empty base but cascades whenever the demanded family already holds indexed inventory. With threshold `0.20`, one `str` waiter, and 25 indexed `str` slots all reserved, successive ticks walk `0/25 → 1/26 → 2/27 → …` and do not clear the threshold until the seventh page. In general, from `(f, t)` the fixed point requires

  ```
  k ≥ (threshold · t − f) / (1 − threshold)
  ```

  which is unbounded in `t`: the larger the existing indexed inventory, the more pages a single waiting field provokes.

The cascade is structural, not arithmetic clumsiness. A ratio target converges only if each provisioning step grows the numerator by a *fraction of the population*, but the emitted index set is sized by **demand** — one waiter yields one indexed column. Making a ratio converge would require indexing far more columns than anyone is waiting for, which [`implementation_phases.md`](../implementation_phases.md) Phase 2 forbids: "a composite B-tree index … is emitted for each slot column named in the provisioner's filterable-column set (the filterable fields awaiting slots); all other columns on the page are created unindexed." ADR `0003` makes the same commitment — index provisioning is schema-driven, never speculative.

So the two constraints are jointly unsatisfiable as stated: *usable capacity as a ratio threshold* and *no speculative indexing* cannot both hold. One of them has to give, and it should not be the no-speculative-indexing rule, which is load-bearing for write cost and for ADR `0012`'s monotonic-page-growth budget.

## Decision

**Usable capacity governs provisioning as a per-family satisfiability test, not as a ratio threshold.** The Watcher provisions when either of two OR-composed triggers fires:

```
Trigger 1 — unsatisfiable demand:
    ∃ family f with waiters where indexedFree[f] = 0   →  provision, ignoring the threshold
Trigger 2 — capacity headroom:
    globalFreeRatio() < threshold                      →  provision
```

Where `indexedFree[f]` counts free slots of family `f` whose column carries an index on its page, and a *waiter* is a filterable field with no live slot — the union of ADR `0016`'s two named demand sources, which collapse into one query because a field with a `running` retype checkpoint provably has no live slot.

The reported trigger prefers Trigger 1: starvation is the actionable condition and must not be reported as routine headroom.

### Why this satisfies AC#1's guarantee

AC#1's operative promise is that *"a page full of unindexed free columns never masks a real shortage of indexed capacity."* Trigger 1 delivers exactly that, deterministically: it never looks at unindexed slots, so no quantity of unindexed free inventory can suppress it. And because the two triggers are OR-composed, this rule is strictly **more** eager to provision than any ratio formulation — anything a usable-capacity ratio would have caught is still caught. The guarantee therefore holds a fortiori.

### Termination

Trigger 1 fires only when a demanded family has nothing claimable. The provisioned page carries at least one indexed free column of every demanded family (the index-set floor below), so `indexedFree[f] ≥ 1` immediately afterwards and the trigger clears. **One page per starved family-set, with no cascade.** Trigger 2 is the pre-existing, empirically stable behaviour, unchanged.

Starvation-freedom follows directly: a waiter of family `f` starves only if `indexedFree[f] = 0` persists across ticks, and that is precisely the condition Trigger 1 fires on. Multiple waiters of one family serialise — each reservation returns `indexedFree[f]` to zero and re-arms the trigger.

### Index-set sizing

For each demanded family, the new page indexes

```
k[f] = clamp( waiters[f] − indexedFree[f],  1,  familyCapacity(f) )
```

columns, taken in declaration order. The **floor of 1** is required by AC#1's unconditional wording ("must carry an index on at least one free column of that family"), and binds the Trigger-2 path too, where the shortfall may be zero or negative. The **cap** is the family's per-page capacity. With no demand at all the set is empty and the page is pure headroom — indexing speculatively is what Phase 2 and ADR `0003` forbid.

### The ratio survives as an observable

`usable_free_slots`, `usable_total_slots`, and `usable_free_ratio` are computed and emitted on every poll cycle, satisfying AC#6's "usable capacity percentage". They are **diagnostics, not triggers.** When demand is empty the usable ratio reports the global free ratio, because with nobody waiting every free slot is usable by whatever arrives next; reporting `0/0` would otherwise make an idle deployment provision on every tick.

## Consequences

**Positive:**

- Provisioning terminates. One waiting field costs at most one page, whatever the size of existing inventory — the property no ratio formulation has.
- The guarantee AC#1 actually cares about is enforced by construction rather than by arithmetic that happens to come out right, and it is trivially auditable: a family either has a claimable slot or it does not.
- The policy is a pure function of `(capacity snapshot, demand, threshold)`, so the entire matrix — including the pathological cases that motivated this ADR — is unit-testable without a database.
- No new configuration. `watcherCapacityThreshold` keeps its existing meaning and arithmetic; what changes is that it is no longer the *only* trigger.
- Deferred retype and promotion reservations (ADR `0016` commitment 4) become genuinely satisfiable by Watcher-provisioned capacity, which they were not while the Watcher emitted unindexed pages.

**Negative:**

- **`threshold = 0.0` no longer means "never provision."** Trigger 1 ignores the threshold by design — that *is* the starvation-freedom guarantee. Deployments and tests that used `0.0` to disable provisioning must instead ensure there is no pending demand.
- The trigger is binary per family, so it carries no notion of *how close* a family is to starvation. A family down to its last claimable slot provisions only once that slot is taken, not before. Acceptable because the Watcher's poll interval is short relative to reservation rate, and because Trigger 2 still provides population-level headroom.
- Usable capacity is now reported but not acted on, which is a mild asymmetry an operator could find surprising. Mitigated by documenting it at both the ADR and code level; the alternative — acting on it — is precisely what this ADR rejects.
- Indexedness is derived per poll from `information_schema.STATISTICS` because the registry does not persist the emitted index set. Correct but not free; see Related for the future schema change that would remove the dependency.

**Rejected alternatives:**

- **Usable capacity as a ratio, with either denominator** — the decision this ADR exists to overturn. Both diverge; proofs in Context.
- **Sizing the index set to clear the threshold** (`k[f] = ⌈(threshold·indexedTotal[f] − indexedFree[f])/(1 − threshold)⌉`) — the only way to make a ratio converge. It emits roughly seven indexed `str` columns for one waiting field, contradicting Phase 2 and ADR `0003`, and spends write amplification on columns no registered field has asked for.
- **Indexing every column on every page** — removes the problem entirely (all free capacity becomes claimable) at the cost of 60 index trees per page maintained for every row written. A large, permanent write-path tax to avoid a bounded planning decision, and it makes ADR `0003`'s schema-driven provisioning vacuous.
- **A second configuration knob for a usable-capacity threshold** — two settings governing one policy, free to disagree, and it would not fix the divergence that motivated this ADR.

## Related

- ADR `0016` — Field Type Change Lifecycle (the clarification this ADR resolves; its deferred-wait guarantee is what Trigger 1 makes good on)
- ADR `0003` — Schema-Driven Index Provisioning (the no-speculative-indexing commitment that makes a converging ratio impossible)
- ADR `0034` — Non-Filterable Fields Are JSON-Only (why every pending reservation now requires an indexed slot, reducing demand shape to the slot family)
- ADR `0008` — Singleton Watcher with Multi-Worker Reconciler (the daemon whose trigger this governs)
- ADR `0012` — Immutable Extension Page DDL (why a page's index set is fixed at creation, and why redundant pages are a real cost)
- ADR `0007` — Write Availability Over Query Completeness (the exhaustion fallback whose waiters are one of the two demand sources)
- ADR `0019` — Index Cardinality Policy (shares the Watcher's poll cycle)
- ADR `0020` — Structured Logging Mandate (the poll and provision payloads widened here; no new event names)
- ADR `0031` / `0032` — Slot Spread Metric, Model-Affine Slot Reservation (both will want demand projected by model; the demand DTO is shaped to allow that additively)
