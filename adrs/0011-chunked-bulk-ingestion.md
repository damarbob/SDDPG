# 0011 - Chunked Bulk Ingestion with Per-Chunk Transaction Scoping

**Status:** Proposed
**Created:** 2026-04-18

## Context

StarDust supports high-volume multi-row ingestion through the `EntriesManager`. A bulk import may contain thousands or tens of thousands of entities submitted in a single API call. Each entity requires writes to `entry_data` (core payload), one or more `entry_slots_page_X` tables (extension slots), and potentially the `stardust_sync_queue` (if slots are exhausted).

Wrapping the entire bulk batch in a single database transaction provides whole-batch atomicity — either every entity commits or none do. However, a single transaction spanning thousands of multi-table writes has severe operational consequences on InnoDB: prolonged row-level locks block concurrent reads and writes on the affected rows, the undo log grows proportionally to the transaction size risking OOM conditions, and replication lag accumulates because the binlog event is not flushed until the transaction commits. On a large enough batch, a single transaction can stall the entire database for seconds or minutes.

The alternative — no transaction boundaries at all (auto-commit per row) — provides no atomicity guarantees even within a single logical entity. An entity that writes successfully to `entry_data` but fails on its extension table write leaves the system in an inconsistent state that the Reconciler is not designed to recover from, because the entry was never enqueued.

## Decision

The `EntriesManager` chunks high-volume bulk ingestion into configurable batches (default: 500 entities per chunk). Each chunk is wrapped in its own database transaction. If a chunk commits successfully, its entities are durable. If a chunk fails, only that chunk is rolled back; previously committed chunks are not affected.

Transactions are scoped to the chunk, not to the entire batch or to individual rows. Within a chunk, the full write sequence — `entry_data` insert, extension table upsert, and optional queue enqueue — is atomic. Across chunks, no atomicity guarantee exists: a failure mid-batch leaves some chunks committed and others uncommitted.

### Synchronous vs. Asynchronous Submission

Chunked transactions solve InnoDB safety, but a multi-minute synchronous HTTP request introduces a second failure mode: the consumer's connection (or any reverse proxy in front of the API) can drop mid-batch. When that happens, the server has no way to deliver the per-chunk success/failure manifest, and the consumer cannot determine which chunks committed without re-reading every entity by external ID. The mitigations below split bulk ingest by size to keep the synchronous path honest and route truly large batches through the same async-job pattern as `/api/exports` (ADR `0010`).

- **Synchronous threshold: ≤ 1,000 entities per request.** Below this size, `POST /api/entries` accepts the batch, processes it inline using the chunked-transaction model above, and returns a single response that enumerates per-chunk outcomes. The expected end-to-end latency is bounded such that a typical reverse-proxy timeout (60s default) will not trigger.
- **Async-required threshold: > 1,000 entities per request.** Requests exceeding the synchronous threshold MUST be submitted to `POST /api/entries/bulk`, which validates the payload, persists it to a job record (artifact path on local disk, identical to the export pattern), responds `202 Accepted` with an Import Job ID, and enqueues the job for the **Reconciler** to process — no separate "Importer" daemon is introduced. Submitting a > 1,000-entity batch directly to `POST /api/entries` returns `413 Payload Too Large` with a pointer to the bulk endpoint.
- **Status and manifest polling.** `GET /api/entries/bulk/{job_id}` returns the job's current status (`pending | processing | completed | failed`) and, for jobs that have produced any chunks, a manifest enumerating each chunk's outcome (`committed | rolled_back | failed`), the entity ID range, and any failure reason. The manifest is durable in the job row and survives consumer reconnects — partial-success state never depends on holding an HTTP connection open.
- **Idempotency key.** Both endpoints accept an optional `Idempotency-Key` header. A retry with the same key returns the original job's status and manifest rather than re-processing — the consumer can safely retry on connection failure without risking duplicate ingestion.

The threshold value is configurable per deployment but the **shape** of the contract is not: the synchronous endpoint MUST refuse oversized payloads, and the async endpoint MUST persist the manifest before responding with `202`. Implementations that quietly accept oversized payloads on the synchronous endpoint reintroduce the dropped-connection failure mode this ADR is designed to eliminate.

## Consequences

**Positive:**

- InnoDB lock duration is bounded by chunk size, not by total batch size. A 500-entity chunk holds row locks for milliseconds, not the seconds or minutes that a 50,000-entity transaction would require.
- Undo log growth is bounded per chunk, eliminating OOM risk from large bulk imports.
- Replication lag is bounded per chunk. Each chunk's binlog event flushes independently, preventing replication stalls on replicas.
- Partial progress is preserved on failure. If chunk 47 of 100 fails, chunks 1–46 are already committed. The consumer can retry from the failure point rather than re-submitting the entire batch.

**Negative:**

- Bulk writes are not atomic at the batch level. A failure mid-batch leaves the system in a state where some entities are committed and others are not. Consumers must design their retry and idempotency logic to handle partial completion.
- Consumer-facing error responses must distinguish between "entire batch failed" and "batch partially committed — these chunks succeeded, these did not." This adds complexity to the API contract.
- Chunk boundaries are arbitrary with respect to the consumer's data. Entities that the consumer considers logically related may land in different chunks, and one may commit while the other rolls back.
- Two ingestion endpoints (synchronous `/api/entries` and async `/api/entries/bulk`) must be documented and versioned together. Consumers integrating bulk import must understand the size threshold, the `413` response, and the polling contract — a heavier learning curve than a single endpoint.
- The Reconciler now drains two distinct workloads (capacity-exhaustion sync queue and bulk import jobs). Operators must monitor both queue depths separately to attribute backlog correctly.

**Rejected alternatives:**

- Single transaction wrapping the entire batch — unbounded lock duration, undo log growth proportional to batch size, and replication stalls. A 50,000-entity batch in a single transaction can hold row locks for minutes and generate gigabytes of undo log.
- No transaction boundaries (auto-commit per row) — no atomicity guarantees even within a single entity's multi-table write sequence. A failure between the `entry_data` insert and the extension table upsert leaves the entity in an unrecoverable inconsistent state.
- Savepoints within a single outer transaction — provides per-chunk rollback semantics but the outer transaction still grows unboundedly. The undo log, lock duration, and replication lag problems are not solved because the outer transaction does not commit until the entire batch completes.
- Synchronous-only with arbitrary payload size — leaves the dropped-connection failure mode unaddressed. Even with bounded chunks, a 60-second batch is statistically guaranteed to drop occasionally on a busy reverse proxy, leaving the consumer blind to which chunks committed.
- Dedicated "Importer" daemon parallel to the Reconciler — duplicates the Reconciler's existing chunked-write infrastructure (configurable chunk size, throttle, `SKIP LOCKED` claim semantics) without benefit. The Reconciler is already designed to drain queued work into extension tables; routing bulk imports through it reuses that machinery rather than duplicating it.
