# 0015 - Database as Sole Daemon Coordination Point

**Status:** Proposed
**Created:** 2026-04-18

## Context

StarDust operates three independent background daemons — the Watcher (capacity provisioner), the Reconciler (queue drain and backfill), and the Liberator (slot eviction and capacity reclamation). These daemons have interdependencies: the Reconciler needs to know when new capacity is available (provisioned by the Watcher); the Watcher should be aware of tombstoned slots that the Liberator has not yet swept; the Liberator must not evict slots that the Reconciler is actively backfilling.

In conventional distributed systems, daemon-to-daemon coordination is accomplished through message queues (RabbitMQ, Redis Streams), pub/sub channels, shared filesystems, or direct IPC (HTTP callbacks, Unix sockets). Each of these introduces an additional infrastructure dependency and a new failure domain — if the message broker goes down, daemons cannot coordinate; if a callback endpoint is unavailable, the notifying daemon must handle the failure.

The zero-dependency core (ADR `0002`) establishes that the system runs on MySQL alone without requiring external infrastructure. This constraint applies to the daemon coordination layer as well: introducing Redis or RabbitMQ purely for daemon signaling would violate the architectural guarantee that the system can be deployed on a single MySQL instance with no additional services.

## Decision

All daemon coordination occurs exclusively through database state. The schema registry tables are the sole communication channel between the Watcher, Reconciler, and Liberator. No message bus, pub/sub channel, shared filesystem, or direct IPC mechanism exists between them.

Specifically:

- **Watcher → Reconciler**: The Watcher's atomic registry update (marking a new page as available) is the sole signal the Reconciler consumes to discover newly provisioned capacity. The Reconciler polls the registry on each cycle — no push notification is sent.
- **Watcher → Liberator**: The Watcher checks registry state for tombstoned slots when calculating available capacity. No direct communication occurs.
- **Liberator → Watcher**: The Liberator's registry update (marking a swept slot as `free`) is discovered by the Watcher on its next polling cycle. The freed capacity is factored into the Watcher's provisioning threshold calculation.
- **Reconciler ↔ Liberator**: No direct interaction. The Reconciler operates on the `stardust_sync_queue`; the Liberator operates on tombstoned registry entries. Their work is disjoint by design.

Daemon responsiveness is limited to polling intervals. The Reconciler does not learn about new capacity until its next poll cycle after the Watcher commits the registry update. This introduces coordination latency proportional to the polling interval (configurable, not instantaneous).

### Schema Cache Invalidation

The synchronous ingestion path (the API workers, not the daemons) caches the schema registry to avoid a multi-table lookup on every write. A naive static TTL creates a forced trade-off between cache effectiveness and recovery responsiveness: too long, and the API stays blind to newly provisioned pages during an exhaustion burst, needlessly funneling writes into `stardust_sync_queue`; too short, and the API hammers the registry tables on every request.

The cache invalidation strategy is **registry-versioned, not time-versioned**:

- **A `stardust_schema_version` row** (or a `schema_version` column on a singleton control table — implementation detail) holds a monotonically increasing integer. Every transaction that mutates the registry's coordination-relevant state — page provisioning (Watcher), slot status transitions (all daemons), field metadata changes — increments this value in the same transaction as the underlying mutation. The increment is cheap; it shares the registry transaction's commit flush.
- **The API cache key includes the version.** Each cache entry is keyed by `(schema_version, lookup_key)`. A stale cache entry is harmless: it simply yields a cache miss when the version advances, prompting a fresh registry read.
- **Version-only liveness probe.** On every request, the API issues a single small `SELECT version FROM stardust_schema_version` (a single-row, single-column read on a tiny table — covered by the buffer pool, sub-millisecond). If the value matches the cached version, the cached registry snapshot is reused. If it differs, the snapshot is reloaded.
- **Bounded staleness fallback TTL.** As a defense against pathological scenarios (e.g., a buggy mutation that fails to bump the version), every cache entry also carries a hard 60-second TTL. The TTL acts as a safety net, not as the primary refresh mechanism.

This design eliminates the cache-stall window (the API picks up new capacity within one request of the Watcher's commit) without bypassing the cache (writes still amortize across cache hits between version bumps). It introduces no new coordination channel — the version row is itself part of the registry, so it is governed by the same "database as sole coordination point" rule.

## Consequences

**Positive:**

- No additional infrastructure is required. The system deploys on a single MySQL instance with no message broker, cache layer, or coordination service. This preserves the zero-dependency core and minimizes the operational surface.
- Each daemon is a fully independent failure domain. If one daemon crashes, the others continue operating with the last known registry state. No daemon's availability depends on another daemon's liveness.
- The coordination model is observable and debuggable. All daemon state is queryable via standard SQL against the registry tables. Operators can inspect the coordination state without specialized tooling.
- The coordination model is inherently consistent — all daemons read from the same transactional database, so there are no eventual-consistency windows between a write and its visibility (within a single MySQL instance).

**Negative:**

- Daemon responsiveness is bounded by polling intervals. The Reconciler cannot react to a Watcher provisioning event faster than its poll cycle, introducing latency in the recovery path after slot exhaustion.
- Polling introduces unnecessary database load during quiet periods when no coordination events have occurred. Each daemon executes registry queries on every cycle regardless of whether state has changed.
- The pattern does not scale to a large number of coordinating daemons. With three daemons on configurable polling intervals, the load is negligible. If the daemon count grew significantly, polling-based coordination would require optimization (e.g., conditional polling, exponential backoff).

**Rejected alternatives:**

- Message queue (RabbitMQ, Redis Streams) for daemon-to-daemon notification — violates the zero-dependency core (ADR `0002`). Introduces a new failure domain: if the message broker is unavailable, daemons cannot coordinate, and the system's resilience guarantees degrade.
- Filesystem-based signals (lock files, named pipes) — brittle and platform-dependent. Named pipes block on read/write, complicating daemon lifecycle management. Lock files require cleanup on crash and are unreliable across networked filesystems.
- Direct IPC (HTTP callbacks, Unix sockets between daemons) — couples daemon lifecycles. If the Watcher notifies the Reconciler via HTTP callback and the Reconciler is unavailable, the Watcher must handle the failure, implement retries, and manage a notification queue — effectively reimplementing a message broker within the daemon itself.
- Static schema-cache TTL only (no version row) — rejected because it forces an unwinnable trade-off between recovery responsiveness during exhaustion bursts (favors short TTL) and registry read amortization (favors long TTL). The version-row probe makes both fast in the common case.
- Bypass the schema cache entirely — rejected because every write would issue a multi-table registry lookup, multiplying registry read load by the request rate and making the registry tables a hot contention point under load.
- MySQL `LISTEN/NOTIFY`-style push (e.g., via the binlog or a polling-on-binlog mechanism) — rejected as over-engineered for the coordination needs at hand. The version-row probe is one trivial query per request and requires no special replication tooling, plugin, or CDC consumer.
