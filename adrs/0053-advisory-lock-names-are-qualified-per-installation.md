# 0053 - Advisory Lock Names Are Qualified Per Installation

**Status:** Proposed
**Created:** 2026-09-16

## Context

ADR [`0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md) made cron-only shared hosting a supported deployment target, and `docs/deployment.md` now walks an operator through a cPanel-style account with a crontab. Shared hosting has a defining property that none of the engine's prior deployment targets had: **many unrelated accounts share one `mysqld`.** Every ADR before this one was written against a deployment where the database server belonged to the installation.

The engine takes exactly two MySQL advisory locks, and both names were fixed literals:

- `GET_LOCK('stardust_page_provision', 10)` — the Watcher's provisioning exclusion, normative in [`blueprints/watcher_reconciler_daemons.md`](../blueprints/watcher_reconciler_daemons.md) AC#2.
- `GET_LOCK('stardust_sweep_page_{pageId}', 0)` — ADR [`0049`](0049-multi-worker-liberator-excluded-per-page.md)'s per-page Liberator exclusion.

**Measured on MySQL 8.0.13: `GET_LOCK` names are scoped to the server, not to the database.** Two sessions connected to two different schemas of one instance, both asking for `stardust_page_provision`: the first acquired it, the second blocked for its full 2000 ms timeout and returned `0`. `performance_schema.metadata_locks` showed one `USER LEVEL LOCK` row, not one per schema. The same held for `stardust_sweep_page_1`.

So two StarDust installations on one shared host excluded each other's Watcher and each other's Liberator sweeps. The page-lock case is the worse of the two, because **page ids restart at 1 in every installation** — `stardust_sweep_page_1` is not merely a possible collision, it is the name every installation reaches for first.

**This is a liveness defect, not a correctness one, and the distinction is load-bearing for how much it is worth.** Neither collision can corrupt anything: the Watcher catches `AdvisoryLockTimeoutException`, logs `lock_contention` and skips provisioning for that tick, and the Liberator's zero-timeout acquisition skips the page and retries next cycle. Both recover on their own. What they cost is real, though, and concentrated in exactly the deployment mode that introduced the problem:

- A Watcher can spend `watcherProvisionLockTimeoutSeconds` (default 10) of a cron tick's 50-second budget blocked on a stranger's lock — a fifth of the run, doing nothing.
- Sustained contention starves provisioning. That is the one failure mode `docs/deployment.md` and the README both single out by name: capacity is never replenished, and new filterable writes fall back to the unindexed JSON payload (ADR [`0007`](0007-write-availability-over-query-completeness.md)).
- An operator debugging it has nothing to go on. The contention is invisible from inside the account — another tenant's lock does not appear in any view they can read.

## Decision

**Both advisory lock names are qualified with a per-installation suffix, derived by default from the schema name.**

1. **A new `Daemon\LockNamespace` owns the qualification.** `qualify(string $baseName): string` returns `$baseName . '_' . $suffix`, where `$suffix` is the first 12 hex characters of `sha1()` of the discriminator. It is the single place a lock name is built; no caller assembles one.

2. **The default discriminator is `DATABASE()`.** StarDust has no table-name prefix — `entry_data` and the `stardust_*` registry tables are fixed names — so one schema holds at most one installation, which makes the schema name a sound identity for it. This is what makes the fix work with no configuration: an operator who has never heard of this ADR is covered.

3. **`Config::$lockNamespace` (nullable string, appended last) overrides it.** Two cases the default cannot serve: an installation that must keep one lock identity across a database rename, and two installations that must deliberately share one. It is a discriminator, not a switch — there is no value that restores unqualified names, because a per-installation lock is the correct behaviour and a mode that turns it off is a mode to support forever.

4. **The base name and the timeout both stay normative.** Blueprint AC#2's `stardust_page_provision` survives as a *prefix* of what reaches the server, and its `10` is untouched. This ADR changes who a lock excludes, not what it means or how long a caller waits for it. Keeping the base greppable is deliberate: an operator reading `performance_schema.metadata_locks` can still tell what a held lock is for.

5. **The suffix is hashed rather than appended raw, for a length reason that is arithmetic rather than taste.** A schema name may be 64 characters and `GET_LOCK` rejects a name over 64. Twelve hex characters leaves the longest name the engine can build — `stardust_sweep_page_` plus a 19-digit `BIGINT` plus a separator plus the suffix — at 52 characters. The budget cannot overflow, so no runtime length check exists and none is needed.

6. **`null` leaves a name unqualified, and that is a test seam, not a deployment option.** `Watcher` and `SweepPageLock` both accept `?LockNamespace` defaulting to `null`, which preserves the literal. `StarDust::watcher()` and `StarDust::liberator()` always pass a real one, so no supported entry point produces an unqualified name. The seam exists because the pre-0053 fixtures assert on literals and should keep doing so.

7. **`StarDust::lockNamespace()` memoises the instance.** Resolving the default costs a `SELECT DATABASE()`, and `combinedTick()` constructs both lock-taking daemons in one process; memoising keeps that at one round trip per process rather than one per daemon.

### New events

None. No event name, field or taxonomy changes — ADR [`0020`](0020-structured-logging-mandate.md) is untouched. `lock_contention` keeps its name and meaning; it simply stops firing for a neighbour's lock.

## Consequences

**Upgrading is not zero-downtime, and this is the one operational cost.** A qualified name and an unqualified one do not exclude each other, so during a rolling deploy a Watcher on the old code and one on the new can provision concurrently, and two Liberators can sweep one page at once — precisely the conditions ADR 0049's page lock exists to prevent. Neither is silently destructive (page provisioning is `CREATE TABLE IF NOT EXISTS` plus a transaction; concurrent same-page sweeps produce lock contention and the ADR [`0046`](0046-a-gapped-sweep-skips-the-chunk-and-does-not-reclaim.md) gap path, which fails closed), but neither is intended either. **Stop the daemons before deploying rather than rolling them.** On a cron-only host that is one skipped tick. This is acceptable to state rather than engineer around because the tag is `0.3.0-alpha.1` and no compatibility promise is outstanding; a straddling release that acquired both names would cost a permanent second acquisition for a one-deploy window.

**The Watcher's cross-host singleton guarantee narrows in one specific way that is worth naming.** ADR [`0027`](0027-persistent-process-daemon-execution-model.md) makes the PID file primary enforcement and `GET_LOCK` the safety net, but a PID file is per-host — so across two hosts, the advisory lock *is* the excluder. Qualifying the name does not weaken that (both hosts resolve the same schema to the same suffix); it only stops the net catching a different installation's Watcher, which it was never meant to catch.

**A shared-hosting operator gains nothing observable when this works.** There is no new event and no new field; the improvement is the absence of `lock_contention` records that were never theirs. That is the right shape for this fix, but it means the deployment docs carry the explanation, since nothing in the event stream will.

**`Config` gains its first field whose default is resolved outside `Config`.** Every prior optional field resolves to a literal in the constructor; this one stays `null` there and defers to `LockNamespace`, because deriving it requires a `SELECT DATABASE()` and `Config`'s constructor performs no I/O of any kind today. The resolution point moving out of `Config` is deliberate rather than an inconsistency — the alternative is a config object that touches the database to construct itself.

## Verification

- **The premise was measured, not assumed**, per the project's standing rule that a load-bearing MySQL claim gets probed against a real server. Two PDO sessions on two schemas of one 8.0.13 instance: `A GET_LOCK('stardust_page_provision', 10) -> 1`, `B GET_LOCK('stardust_page_provision', 2) -> 0 after 2005ms`, and the same result for `stardust_sweep_page_1` at a zero timeout. The 2005 ms is the part that rules out a fast-fail misread — B genuinely waited out its timeout.
- `tests/Smoke/Daemon/LockNamespaceTest` — six cases over two sibling connections, which is the only way to see this: `GET_LOCK` is re-entrant within one session, so a single-connection test cannot distinguish "the names differ" from "this session already holds it."
- **Both directions are asserted deliberately.** That two namespaces stop excluding each other is the fix; that one namespace still excludes is ADR 0049's guarantee the fix must not cost. A test for only the first would pass if `qualify()` returned a random string per call.
- **Neutered validation**: `qualify()` was temporarily changed to return the base name unqualified. Three of the six went red, including the behavioural one — `testSweepPageLockIsPerInstallation` failed on the second installation's page-1 acquisition returning `null`, which is the defect itself reproduced. The three that stayed green are the unqualified-literal seam and the length-budget assertion, which is correct for a neuter that removes qualification.
- Full smoke suite green and PHPStan level 8 clean with no baseline.

## Related

- [ADR `0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md) — Made shared hosting a supported target and so created the condition this ADR fixes. Every prior ADR assumed the database server belonged to the installation.
- [ADR `0049`](0049-multi-worker-liberator-excluded-per-page.md) — Introduced `stardust_sweep_page_{pageId}`. Its page-ids-restart-at-1 property is what makes that the most collision-prone name in the engine; this ADR qualifies the name and leaves the granularity decision entirely intact.
- [ADR `0027`](0027-persistent-process-daemon-execution-model.md) — PID file primary, `GET_LOCK` the safety net. This ADR narrows what the net catches (never another installation) without changing the model.
- [ADR `0026`](0026-framework-neutral-composer-packaging.md) — The framework-neutral packaging that makes an injected `PDO` the engine's only database handle, which is why `LockNamespace` resolves its default off that same connection rather than parsing a DSN.
- [ADR `0007`](0007-write-availability-over-query-completeness.md) — The JSON-payload fallback that starved provisioning degrades into, which is what makes a liveness defect here worth fixing rather than tolerating.
- [`blueprints/watcher_reconciler_daemons.md`](../blueprints/watcher_reconciler_daemons.md) — AC#2 specifies the literal `stardust_page_provision`. The base name survives as a prefix and the 10-second timeout is unchanged; the blueprint needs a note that the name reaching the server carries an installation suffix.
