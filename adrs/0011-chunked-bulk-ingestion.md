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

**Rejected alternatives:**

- Single transaction wrapping the entire batch — unbounded lock duration, undo log growth proportional to batch size, and replication stalls. A 50,000-entity batch in a single transaction can hold row locks for minutes and generate gigabytes of undo log.
- No transaction boundaries (auto-commit per row) — no atomicity guarantees even within a single entity's multi-table write sequence. A failure between the `entry_data` insert and the extension table upsert leaves the entity in an unrecoverable inconsistent state.
- Savepoints within a single outer transaction — provides per-chunk rollback semantics but the outer transaction still grows unboundedly. The undo log, lock duration, and replication lag problems are not solved because the outer transaction does not commit until the entire batch completes.
