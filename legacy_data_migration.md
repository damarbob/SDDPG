# Legacy Data Migration

> **Status:** Stub — operational details deferred.
>
> **Prerequisites for re-authoring:**
>
> 1. The Architecture Blueprint reaches a stable baseline. Current cadence is weekly material change, which renders operational specifics obsolete on the order of days.
> 2. The legacy **Virtual Column Method** is documented to a level that lets a contributor reason about source-side semantics — schema shape, write path, consistency guarantees, known footguns.
>
> Until both prerequisites are satisfied, this document records only the load-bearing intent. The principles below are normative for any future migration plan because they are properties of the _transition_, not the destination — they survive blueprint churn. Previously documented specifics (DLQ thresholds, command names, gate sample sizes, message queue choice, replay tooling) have been removed: being specifically wrong is worse than being honestly absent.

## Context

StarDust is migrating from a legacy **Virtual Column Method** (single-table, generated-virtual-column indexing) to **Vertical Schema Partitioning** (`entry_data` + `entry_slots_page_X`; see [`architecture_blueprint.md`](architecture_blueprint.md)). The migration must execute without locking or threatening availability of the legacy system.

## Load-Bearing Principles

### 1. Synchronous atomic dual-writes are forbidden

The API path writes only to the legacy system synchronously. Replication into the new schema is asynchronous and event-driven. This bounds the latency cost of the migration to the legacy path's performance and guarantees the new schema can never block legacy availability.

### 2. The consumer reads the authoritative payload, not the event snapshot

The consumer worker must read the authoritative source (`entry_data.fields` for the new schema; legacy-equivalent for the source side) at upsert time, never the event payload that triggered it. Stale backfill events must not overwrite fresher real-time updates for the same entry. See [`adrs/0013-json-payload-as-system-of-record.md`](adrs/0013-json-payload-as-system-of-record.md).

### 3. Cutover is gated, not flagged

Cutover proceeds through a sequence of quantifiable gates — stream-drain, data-parity, optional shadow-traffic. A single boolean "switch traffic to the new schema" flag is insufficient. Each gate is a precondition that must be independently verified before the next is attempted.

### 4. Read rollback and write replication are decoupled

The flag controlling which schema serves reads is independent of the flag controlling whether the event producer is active. If reads roll back to legacy, the producer **must remain active** so the new schema continues shadowing legacy state. Re-cutover must never require a multi-hour cold backfill.

## What This Document Does Not Specify

The following were previously specified here and have been intentionally removed pending re-authoring:

- DLQ depth and age alerting thresholds
- CLI command names for replay, backfill, and checkpoint inspection
- Backfill checkpoint table shape
- Gate sample sizes, drain durations, shadow-traffic percentages
- Message queue technology choice (Kafka, Redis Streams, SQS, etc.)
- Per-entry idempotency key strategy
- Rollback acceptance criteria

These will be authored when the prerequisites above are satisfied.

## Related Documents

- [`architecture_blueprint.md`](architecture_blueprint.md) — destination architecture.
- [`adrs/0007-write-availability-over-query-completeness.md`](adrs/0007-write-availability-over-query-completeness.md) — write-availability principle that informs the async dual-write decision.
- [`adrs/0013-json-payload-as-system-of-record.md`](adrs/0013-json-payload-as-system-of-record.md) — payload-authority principle that informs the freshness guarantee.
