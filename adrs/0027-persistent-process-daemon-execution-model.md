# 0027 - Persistent-Process Daemon Execution Model

**Status:** Proposed
**Created:** 2026-05-14

## Context

StarDust's runtime depends on four long-running background processes — the Watcher, Reconciler, Liberator, and Chronicler ([`architecture_blueprint.md`](../architecture_blueprint.md) §2.1). [ADR `0026`](0026-framework-neutral-composer-packaging.md) ships them as CLI commands (`bin/stardust watcher`, `bin/stardust reconciler`, `bin/stardust liberator`, `bin/stardust chronicler`) but does not specify the execution model under which those commands are expected to run: must they execute as persistent processes (systemd / supervisor units, or long-running containers), or may they be invoked discretely as cron jobs?

The question determines the hosting envelope StarDust — and any CMS depending on it — can be deployed to. Free shared hosting (e.g., InfinityFree-class providers) offers no shell, no cron, and no persistent processes; it is structurally excluded regardless of any other choice. Paid shared hosting with cron jobs (typical cPanel offerings) gives bounded periodic invocation but kills long-running PHP processes via CPU / runtime limits. VPS and container deployments give unrestricted process control.

The architecture is already well-positioned to support a cron-driven execution model with modest engineering work, *because* [ADR `0015`](0015-database-as-sole-daemon-coordination-point.md) externalizes all daemon coordination state to the database. Every daemon checkpoints a durable cursor: the Chronicler via `stardust_export_jobs.last_cursor` combined with the lease / re-claim semantics of [ADR `0025`](0025-chronicler-failure-semantics.md); the Liberator via `stardust_slot_assignments.sweep_cursor_id`; the Reconciler's retype-backfill via `backfill_checkpoints.last_processed_id`; and the Reconciler's primary workload via the sync queue itself. A daemon that exits after one bounded chunk-pass is, structurally, operationally identical to a daemon that crashes and restarts — a scenario the architecture already handles correctly.

The choice is therefore not "is a cron-driven mode feasible?" — it clearly is. The choice is whether StarDust should commit to supporting two execution models, with the ongoing CI, documentation, and operator-support surface that two-model support implies, in exchange for unlocking the paid-shared-hosting deployment tier.

## Decision

StarDust v1 supports a single daemon execution model: **persistent background processes — either OS-level long-running processes (systemd, supervisor, or equivalent) or containerized workers**. The four CLI entry points are expected to run continuously, self-looping their bounded chunk passes internally; they are not designed to be invoked one pass at a time by an external scheduler.

A cron-driven `--once` / single-pass invocation mode is **deliberately deferred, not architecturally foreclosed**. The externalization of all daemon state to the database ([ADR `0015`](0015-database-as-sole-daemon-coordination-point.md)) and the universal cursor-checkpointing pattern across the four daemons mean a future `--once` mode can be added without disturbing any path, contract, or invariant — the architectural cost of adopting it later is small and localized. If a future ADR adopts it, the residual work is mechanical: a per-daemon bounded-pass entry point with a chunk-count or time budget; Chronicler artifact-file append-correctness on resume; and a non-stdout logging sink suitable for cron environments.

### Supported deployment tiers

| Tier | Verdict | Notes |
| :--- | :--- | :--- |
| Free shared hosting (no shell, no cron, no persistent processes) | **Unsupported** | Structurally cannot run the daemon set. |
| Paid shared hosting with cron only | **Unsupported in v1** | Awaits a future `--once`-mode ADR. |
| VPS (systemd, supervisor, or equivalent process supervisor) | **Supported — reference deployment** | Each daemon runs as a managed long-running process. |
| Containerized deployment (Docker Compose, Kubernetes, ECS, etc.) | **Supported — recommended for production at scale** | One container per daemon. The Reconciler scales horizontally per [ADR `0008`](0008-singleton-watcher-multi-worker-reconciler.md); the Watcher MUST be pinned to a single replica, with the in-DB `GET_LOCK` advisory lock as the safety net. The Chronicler requires a shared volume for export artifacts. |

### Host capability requirements (binding)

A supported deployment target MUST provide all of:

1. The ability to run **persistent background processes or long-running containers** (not bounded-runtime invocations such as cron-only shared hosting).
2. MySQL 8.0.13+ or Percona 8.0.13+ ([ADR `0023`](0023-minimum-mysql-version.md)).
3. PHP 8.x with CLI access (required by the `bin/stardust` entry point per [ADR `0026`](0026-framework-neutral-composer-packaging.md)).
4. Local filesystem write access for the Chronicler's export artifacts (a mounted volume in container deployments).
5. PID-file or container-orchestrator-level enforcement of Watcher singleton execution (the in-DB `GET_LOCK` advisory lock is the safety net, not the primary enforcement).

### Host sizing (deliberately not pinned)

Raw RAM / CPU / disk sizing is **not** pinned by this ADR. StarDust's resource footprint is bounded by design — the two-query read path, `LIMIT page_size + 1` enforcement, chunked ingestion, and the lazy-poll daemon profile collectively guarantee a flat memory footprint regardless of dataset size. Resource sizing scales with the tenant's data volume and traffic, not with StarDust's baseline. Concrete sizing guidance belongs in the planned `runbooks/` directory, not in this ADR.

### Documentation Updates

The following documents are updated as part of this ADR landing:

1. [`architecture_blueprint.md`](../architecture_blueprint.md) §1.1 — extends the Operating Environment with a self-contained statement of the daemon execution model requirement, formatted without cross-referencing this ADR (per the README's "blueprint is self-contained" rule).
2. [`README.md`](../README.md) — adds a top-level Deployment Requirements section listing the binding capability requirements above and pointing at this ADR.
3. [`adrs/README.md`](README.md) — adds the index row for this ADR.

## Consequences

**Positive:**

- The deployment story becomes documented and reviewable. Operators, contributors, and future maintainers no longer have to infer the hosting floor from four scattered daemon sections of the blueprint.
- v1 has a single, simple supported execution model. CI tests one mode; documentation describes one mode; runbooks assume one mode. The ongoing maintenance surface stays bounded.
- The recommended deployment (containers) is operationally the cleanest: process boundaries map cleanly to daemon boundaries, the Reconciler's horizontal-scaling property is trivially expressible, and the PID-file / OS-level-process-locking awkwardness from `architecture_blueprint.md` §2.1.4 disappears because the container *is* the process boundary.
- The architecture's resilience design — the exhaustion fallback at `architecture_blueprint.md` §2.1 — is preserved: even if a daemon dies, ingestion never blocks. The persistent-process model does not compromise this.
- The deferral of `--once` mode is explicit and the reasoning is recorded, so a future ADR can revisit it with full context if a real client need emerges.

**Negative:**

- Free shared hosting is excluded permanently from v1's supported set. Operators / clients in that tier must move to a daemon-capable host (a modest VPS at the low end, ~$5/month-class) or wait for a future `--once`-mode ADR.
- Paid shared hosting with cron — a substantial slice of the cPanel-managed hosting market — is also excluded in v1. The cost of cutting this tier is real; the mitigation is the explicit deferral, not foreclosure, of `--once` mode.
- StarSystem (the CMS built on StarDust) inherits this deployment floor in full. Any product built on StarDust inherits the persistent-process requirement and the same hosting envelope.

**Rejected alternatives:**

- **Support both persistent and cron-driven `--once` modes from v1.** Rejected because the build cost is real (per-daemon bounded-pass entry points, Chronicler artifact-file append-correctness on resume, non-stdout logging sinks for cron environments) and the ongoing cost — CI coverage of both modes forever, two operator runbook variants, two debugging paths — is permanently higher than the single-mode baseline. Defer until a specific client need justifies the surface.
- **Foreclose `--once` mode permanently** by adopting in-memory state, message buses, or leader election that would couple daemons to a persistent runtime. Rejected because [ADR `0015`](0015-database-as-sole-daemon-coordination-point.md)'s externalization of state is a load-bearing simplicity property; sacrificing it to harden the single-execution-model choice would be a much larger architectural change with no current motivating need.
- **Loosen the floor to "any host with cron"** without a corresponding `--once` mode. Rejected because shared-host CPU / runtime kill limits would terminate the daemons mid-loop, producing partial work and inconsistent observability. `GET_LOCK` released on connection close partially mitigates lock leaks but is not a substitute for clean termination.

## Related

- [ADR `0008`](0008-singleton-watcher-multi-worker-reconciler.md) — Singleton Watcher vs. multi-worker Reconciler scaling contract.
- [ADR `0015`](0015-database-as-sole-daemon-coordination-point.md) — Database as sole daemon coordination point (the load-bearing property that keeps a future `--once` mode cheap to adopt).
- [ADR `0023`](0023-minimum-mysql-version.md) — Minimum supported MySQL version.
- [ADR `0025`](0025-chronicler-failure-semantics.md) — Chronicler failure semantics (lease / heartbeat / re-claim — already makes export jobs resumable).
- [ADR `0026`](0026-framework-neutral-composer-packaging.md) — Framework-neutral Composer packaging (defines the `bin/stardust` CLI surface this ADR pins the execution model for).
