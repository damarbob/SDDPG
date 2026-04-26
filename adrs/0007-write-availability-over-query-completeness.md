# 0007 - Write Availability Over Query Completeness on Slot Exhaustion

**Status:** Proposed
**Created:** 2026-04-15

## Context

StarDust's extension-table layer has finite capacity at any point in time. Each provisioned page holds a fixed number of typed slot columns, and the total number of pages grows only when the Watcher daemon detects demand and executes the DDL to provision a new one. This means there is always a hard ceiling on how many distinct fields can be simultaneously indexed.

Under sustained high-throughput ingestion — or after a burst of new field mappings — every slot on every provisioned page can be simultaneously occupied: either actively mapped to a live field or held by a tombstoned slot awaiting the Liberator's sweep. When this happens, the slot assignment step of the write path finds no free slot for an incoming field.

At this boundary the system faces a classic availability-vs-consistency trade-off: refuse the write until capacity is restored, or accept it in degraded form and restore full queryability asynchronously. Rejecting the write preserves indexed query completeness for existing data but interrupts ingestion — a hard violation of the throughput guarantees the system was designed to provide. Accepting the write in degraded form keeps ingestion unblocked but temporarily reduces the queryable surface for the affected field.

The blueprint resolves this explicitly in favour of availability: high-throughput ingestion must not block or drop writes. The queue may grow, but writes never block.

## Decision

When the slot assignment step finds no free slot for a field, the write path proceeds without that slot assignment. The core `entry_data` row is written and the full `fields` JSON payload is persisted as normal. Extension table insertion for the unassigned field is skipped for that write only.

The missing slot assignment is recorded in the backfill queue. The Reconciler daemon processes this queue asynchronously: once a free slot becomes available — either because the Watcher provisions a new extension page or because the Liberator completes a tombstone sweep — the Reconciler reads the queued entry, writes the field value into the newly freed slot, and marks the backfill task complete.

During the backfill window, the affected field is still retrievable via `JSON_EXTRACT(fields, '$.fieldName')` from the core payload. This is safe because `entry_data.fields` always holds the complete entry payload; extension table slots are materializations of that payload, not independent stores, so a missing slot never implies missing data. The field is not available as an indexed filter until backfill completes. Once the Reconciler finishes, full indexed queryability is restored without any re-ingestion of source data.

## Consequences

**Positive:**

- Ingestion throughput is never blocked by extension-layer capacity. The write path has a guaranteed forward path regardless of slot availability or DDL provisioning state.
- No data is lost on slot exhaustion. The JSON payload preserves the full entry, and the backfill queue provides a durable record of every slot assignment that needs to be materialised later.
- The failure mode is a graceful, observable degradation — `JSON_EXTRACT` retrieval continues to work — rather than a hard error surfaced to producers.
- Provisioning and ingestion concerns are fully decoupled. The Watcher can provision new pages at a safe, controlled pace without creating write backpressure upstream.

**Negative:**

- During the backfill window, affected fields cannot be used as indexed filters. Consumers may observe incomplete filter results: entries that should match a predicate are absent until their slot assignment is backfilled.
- Under prolonged slot exhaustion, the backfill queue can grow faster than the Reconciler drains it. Queue depth must be monitored as a leading indicator of recovery time.
- Monitoring must distinguish between "write succeeded — slot pending backfill" and "write failed" to avoid masking real failures behind expected degradation.
- The eventual-consistency window for indexed queryability is non-deterministic: it depends on when the Watcher provisions capacity and when the Liberator frees tombstoned slots, neither of which is instantaneous.

**Rejected alternatives:**

- Reject writes with `503 Service Unavailable` until capacity is provisioned — blocks ingestion, violates the write-availability guarantee, and creates upstream backpressure that the system was specifically designed to avoid.
- Synchronously provision a new extension page inline during the write — risks acquiring DDL metadata locks under load, contends with the Watcher's own provisioning cycle, and couples write latency to DDL execution time.
- Buffer writes in application memory pending slot availability — introduces unbounded memory growth, risks data loss on process crash, and moves the queue out of the durable, observable database layer.
