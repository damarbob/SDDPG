# 0052. The Advisory Schedule Is Persisted And Fleet-Wide

**Status:** Proposed
**Created:** 2026-09-16

## Context

ADR [`0019`](0019-index-cardinality-policy.md) gave the cardinality advisory a cadence — "every 24 h, jittered to avoid stampedes" — and ADR [`0031`](0031-slot-spread-metric.md) §Sampling Triggers 1 put the spread advisory on that same timer rather than a second one. The implementation put the timer in a process: `Watcher::$nextAdvisorySampleAt`, a private nullable int, with a `shouldSampleAdvisories()` predicate whose **first call always returns false** — it phase-randomises the first due time as a side effect and samples nothing, which is precisely how a lockstep-started fleet gets spread across the day instead of clumping.

That is correct for a persistent daemon, and inert for the deployment mode ADR [`0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md) introduced. `StarDust::tick()` builds a fresh object graph on every invocation — `combinedTick()` is deliberately not memoised — so a cron-driven host constructs a brand-new `Watcher`, gets the always-false first due-check, and exits. **Neither advisory could ever fire on its own under the only supported mode that has no persistent process.** ADR 0048 shipped `Watcher::sampleAdvisories()` plus a `--advisories` flag as the stopgap, and documented a second, once-daily crontab line as the operator's route to the advisories. The method's own docblock named closing this as separate work rather than letting the workaround become a silent requirement of the mode.

The question this ADR answers is therefore narrow — where does the schedule live — and it has a second half that the narrow framing hides: **persisting it in the database necessarily makes it shared**, which changes the persistent-daemon mode too.

## Decision

### Commitment 1 — the schedule is a dedicated singleton table

`stardust_advisory_schedule`: `id TINYINT NOT NULL`, `next_sample_at DATETIME NULL`, `last_sample_at DATETIME NULL`, `updated_at DATETIME NOT NULL`, `PRIMARY KEY (id)`, `CONSTRAINT ck_advisory_schedule_singleton CHECK (id = 1)`. Created by a new `Bootstrapper::createAdvisorySchedule()` and seeded by a new `seedAdvisoryScheduleSingleton()`, both following the existing patterns exactly — the CHECK is advisory only on 8.0.13–8.0.15 (re-verified on 8.0.13: an `INSERT` of `id = 2` succeeds), so `PRIMARY KEY` plus the seed are the real guarantee, and the seed's `ON DUPLICATE KEY UPDATE id = id` is what stops a re-`bootstrap()` re-randomising a schedule that is already running.

**Not a column on `stardust_schema_version`**, which the internal roadmap named as the alternative and which looks cheaper — one `ensureXxxColumn()` probe, no new table, no test-fixture allowlist entry. Two reasons it is the wrong row. First, it is a global serialization point: `PageProvisioner`, `SlotReserver`, `SlotSweeper` and three Reconciler work sources all `UPDATE ... WHERE id = 1` on it, and ADR [`0008`](0008-singleton-watcher-multi-worker-reconciler.md)'s reasoning against holding it inside a chunk transaction is recorded in the engine's own notes. Second, its `updated_at` means "when the schema last changed"; an advisory claim would have to either write it, making the column lie, or leave it stale beside a column it just changed. A second singleton costs one table and keeps both meanings honest.

**`next_sample_at IS NULL` is the "never scheduled" state**, and it is load-bearing rather than incidental: it is what preserves ADR 0019's first-sample phase randomisation. The first observer writes `now + rand(0, interval)` and deliberately samples nothing.

### Commitment 2 — the schedule is fleet-wide, and this is a behaviour change

One sample fires per interval across the whole deployment, not one per daemon. Three hosts running `bin/stardust watcher` go from three daily sweeps to one.

This is stated as a decision rather than a side effect because it changes shipped behaviour in a mode that was working. It is the right end state — both samplers scan the **global** pool (`CardinalitySampler::sample()` has no tenant or model predicate; `SpreadSampler::sampleAll()` collects every model of every tenant), so N hosts each sampling everything daily was N× duplicate work producing N× the advisory events for one underlying fact. But ADR 0019's stampede rationale was written about per-process schedules, and an operator who counted on per-host sampling would see the count drop. The jitter mechanism is unchanged and still does the job it was added for at the level that now matters: de-correlating **the deployment's** sample from other deployments' and from its own previous one.

### Commitment 3 — the claim is the affected-row count of one UPDATE

```sql
UPDATE stardust_advisory_schedule
   SET next_sample_at = ?, last_sample_at = ?, updated_at = ?
 WHERE id = 1 AND next_sample_at IS NOT NULL AND next_sample_at <= ?
```

`rowCount() === 1` means this caller won and must run the samplers. **The reschedule cannot be split out of the claim** — it is the same statement, because the statement's atomicity is the entire exclusion mechanism. Racing callers serialize on the row lock, and the loser re-evaluates the predicate against the winner's committed row, sees a future `next_sample_at`, and matches nothing.

Verified on MySQL 8.0.13 with two real OS processes before the code was written: the second blocked 1979 ms on the row lock and then reported zero affected rows, with `next_sample_at` advanced exactly once. Verified again end to end with two `StarDust::tick()` runs on **separate `pidFileDir`s**, so the Watcher's own pid-file guard was provably not the excluder — both reported `idle`, and exactly one sampled.

A non-locking `SELECT` runs first and only decides whether a claim is worth attempting, which keeps the common not-due tick to one read. It may be stale; the claim re-evaluates under the lock, so exactness never depends on it.

### Commitment 4 — the next due time is floored at `now + 1`

**MySQL's `UPDATE` affected-row count is *changed* rows, not matched rows**, and the engine takes an injected PDO so it cannot set `CLIENT_FOUND_ROWS`. A claim that wrote back the values already stored would therefore report zero despite matching its predicate, and the sample would be skipped **in silence**. Measured on 8.0.13: an UPDATE matching the row but changing no value reports `rowCount() === 0`.

`Watcher::computeNextDue()` consequently returns `max($from + interval + jitter, $from + 1)`. At the 86 400 s default the floor is unreachable; only a degenerate `interval = 0, jitter = 0` configuration reaches it, and that configuration must still make progress rather than silently stopping.

A related implementation constraint, measured the same way: **a reused *named* placeholder is rejected.** `:now` appearing three times throws `SQLSTATE[HY093]` under native prepares, which an injected PDO may well be using. The claim binds positional `?` with the value repeated.

### Commitment 5 — every datetime is bound from the injected clock

Never `UTC_TIMESTAMP()` in the predicate. The session `time_zone` is `SYSTEM` on a default server, so mixing server time into a comparison against clock-derived stored values would compare two different clocks; and the existing frozen-clock scheduling tests drive the cadence entirely through the injected clock. `UTC_TIMESTAMP()` remains correct in the *seed*, which touches only `updated_at`.

### Commitment 6 — the claim lives in `Watcher::tick()`, so `CombinedTick` is unchanged

`CombinedTick` already calls `Watcher::tick()` once per run unconditionally and again on every `CAPACITY_WAIT` round. Siting the claim inside `tick()` means those repeated calls become no-ops rather than duplicate sweeps, and ADR 0048's composition needed **no modification at all**. The `$advisories` parameter, its `tick_started` field, and the `--advisories` flag all keep their current shapes.

### Commitment 7 — `--advisories` becomes "sample now regardless", and resets the timer

`Watcher::sampleAdvisories()` keeps its unconditional contract and additionally writes the next due time, so a forced sample is not followed minutes later by a scheduled one. The flag is no longer the only route to the advisories under the cron mode, and `docs/deployment.md` no longer presents a second crontab line as required — but forcing a sample on demand is independently useful and the flag stays.

### Commitment 8 — an on-demand CLI for the cardinality advisory

`CardinalitySampler` gains `report(?int $tenantId, ?int $modelId): list<CardinalitySample>` with `trigger='on_demand'`, and `bin/stardust cardinality:report [--tenant=N] [--model=N]` prints it. This mirrors `SpreadSampler::report()` / `spread:report` exactly, including that `report()` both emits and returns so the CLI prints without running the aggregates twice. It closes an asymmetry: the spread advisory had an on-demand reading and the cardinality advisory did not, so an operator between scheduled runs had nothing to reach for.

Three things are deliberately **not** copied from `spread:report`:

- **The safety claim.** `SpreadSampler` is registry-only and documented safe against production at any time. This sampler reads `COUNT(*)` and `COUNT(DISTINCT col)` off every matching extension page. The help text says "read-only, but it scans every matching extension page — run it off-peak" instead.
- **What `--model` means.** It narrows which **slots** are sampled, not which **rows** are counted. ADR 0019's aggregate is per `(tenant, slot)` over the tenant's whole partition on that page, because the index it describes is `(tenant_id, slot_column)` and a page is shared by every model with a slot on it. Narrowing the counts would measure something the optimizer never sees.
- **An INNER join.** `--model` needs `stardust_fields.model_id`, but a tombstoned or grandfathered slot carries `field_id = NULL`. The join is a LEFT join; an INNER one would silently shrink the unfiltered periodic scan with nothing failing.
- **A trusted `--tenant`.** The filter narrows the unfiltered path's `SELECT DISTINCT tenant_id` to a point lookup, but it **confirms** the tenant is present on the page rather than assuming it. Taking the flag's value on trust looks free and is not: a tenant with no rows on a page yields an invented sample at `row_count: 0, distinct_values: 0`, which then trips `low_cardinality_index` on the distinct floor — so a mistyped tenant id would warn about every live slot in the deployment holding none of that tenant's data. Measured during implementation: `report(999)` emitted exactly that. The lookup is still indexed, since `tenant_id` leads every slot's composite index.

`StarDust::cardinalitySampler()` becomes public, as `spreadSampler()` already is, for the same reason.

### Events

**No new event name**, so ADR [`0020`](0020-structured-logging-mandate.md)'s vocabulary is untouched and `EventVocabularyTest`'s allowlist needs no entry. Two additive changes within existing events:

- `poll_complete` (source `watcher`) gains `advisories_sampled` (bool). A lost claim is silent by design — the contended-tick precedent from ADR [`0049`](0049-multi-worker-liberator-excluded-per-page.md) — and this field is the only way to distinguish "not due" from "due but another host won it" on the wire.
- `cardinality_sampled` / `low_cardinality_index` gain `on_demand` as a third value of their existing closed `trigger` set (`periodic` | `post_backfill` | `on_demand`), matching what `spread_sampled` already carries. Recorded in ADR 0019, which is `Proposed` and so editable in place.

## Consequences

- The advisories work unprompted under every supported deployment mode. The cron-only mode's second crontab line becomes optional rather than required, and forgetting it is no longer a silent loss of both advisories.
- **Sample volume drops on multi-host deployments**, from one sweep per host per interval to one per deployment. Intended, and the reason Commitment 2 is a commitment.
- The cadence no longer drifts. The in-memory version rescheduled from `now()` *after* the samplers finished, so each day's sweep pushed the next one later by its own duration; the claim computes the next due time before sampling, since it is the same statement.
- **The claim commits before the samplers run**, so a sampler that throws loses that interval's sample rather than retrying it. This follows unavoidably from Commitment 3 — the claim and the reschedule are one statement, so there is no ordering in which the work precedes the commit — and it is not a regression: the in-memory version also rescheduled only on the success path, but an exception there propagated out of `Watcher::tick()` and killed the process, whose replacement then phase-randomised a fresh schedule and so also waited up to an interval. Both advisories are read-only and purely observational, so the failure mode is a missing observation, not lost or inconsistent state.
- One extra non-locking `SELECT` per Watcher tick, and one `UPDATE` per interval. On the cron mode at one tick a minute that is 1440 reads and one write a day against a single-row table.
- A schedule row deleted out from under a running deployment self-heals: each mutator seeds the singleton first with the Bootstrapper's own no-op upsert, rather than stalling the advisories for ever.
- The engine gains its second singleton table, and `SchemaFixture::CORE_TABLES` its eleventh entry. `Conventions/BootstrapperTableAllowlistTest` enforces the pairing in both directions.
- `Watcher::__construct()` gains an appended nullable `?AdvisoryScheduleRepository`, defaulting to one built from the `PDO` and clock it already holds — so every existing direct construction, including the test fixture's, keeps working unchanged. This is why ADR 0019's three scheduling tests pass **unmodified** against the persisted schedule, which is the regression contract for the phase-randomisation semantics.

## Rejected Alternatives

- **A column on `stardust_schema_version`.** The cheaper option, and the one the roadmap named first. Rejected on the two grounds in Commitment 1: it is a global serialization point, and its `updated_at` has a meaning an advisory claim would have to violate.
- **A row in `backfill_checkpoints`.** No DDL at all, and it already has a `UNIQUE (job_name)` single-row-per-name idiom. Rejected because that table's four namespaces are documented as disjoint thirteen-character prefixes with `LIKE`-escaped claim scans and a `running|paused|completed|failed` ENUM that means nothing for a recurring timer. A fifth, differently-shaped namespace would make a documented invariant false to save one `CREATE TABLE`.
- **Keeping the schedule per-host** (keyed by host identity) to preserve today's persistent-daemon behaviour exactly and fix only the cron case. Rejected: it turns a singleton into an unbounded table nothing prunes, and it preserves behaviour that was duplicate work — N hosts sampling the same global pool.
- **A fleet-wide schedule with no claim**, reading the due time and writing the next one unconditionally. Simpler by one predicate. Rejected as cheap to get right: two hosts ticking in the same second would both sweep, and the claim is one `WHERE` clause.
- **Leaving `CombinedTick` to own the claim** rather than `Watcher::tick()`. Rejected because the Watcher owns the schedule everywhere else — ADR 0020's note that cardinality events carry `source: registry` because "the Watcher merely owns the schedule" is the same division — and because putting it in the tick would leave the persistent daemon on a second, different mechanism.
- **Deriving "due" from `last_sample_at` plus the interval** instead of storing `next_sample_at`. Rejected: the jitter is drawn per cycle, so the next due time is not a function of the last sample and the drawn offset would have to be stored anyway.
- **A new event for a lost claim.** Rejected on the project's own rule that distinct *outcomes* get distinct events while one outcome's causes get a sub-taxonomy — and here the outcome is "this tick did not sample", which `poll_complete` already reports.

## Related

- Amends ADR [`0019`](0019-index-cardinality-policy.md): its cadence is now a persisted, fleet-wide schedule rather than a per-process one, and its `trigger` set gains `on_demand`. Edited in place, as 0019 is `Proposed`.
- Amends ADR [`0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md): the `--advisories` workaround it shipped, and the "run it from a separate once-daily crontab line" instruction, are superseded by Commitments 6 and 7. `CombinedTick` itself is unchanged.
- Extends ADR [`0031`](0031-slot-spread-metric.md) without contradicting it. §Sampling Triggers 1's "one timer, both advisories" rule is preserved exactly — the timer simply moved from a property to a row. 0031 is **`Accepted`**, so it is not edited in place; this pointer is the record.
- Does **not** amend ADR [`0020`](0020-structured-logging-mandate.md) — no new event name. Two additive payload fields within existing events.
- Leaves ADR [`0008`](0008-singleton-watcher-multi-worker-reconciler.md)'s Watcher singleton untouched. The pid-file guard still makes the Watcher a per-host singleton; this ADR adds a database claim *above* it, which is what makes the schedule correct across hosts the pid file cannot see. The two-process verification in Commitment 3 used separate `pidFileDir`s precisely so the pid guard could not be mistaken for the mechanism under test.
