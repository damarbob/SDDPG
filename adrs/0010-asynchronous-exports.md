# 0010 - Asynchronous Exports for Unbounded Data Retrieval

**Status:** Proposed
**Created:** 2026-04-16

## Context

The synchronous read path enforces strict bounding: cursor-based pagination (ADR `0006`), a two-query bounded fetch (ADR `0005`), and pre-flight rejection of unindexed filters (ADR `0004`). These constraints exist to keep every synchronous database operation bounded by page size, not by result-set size. They are non-negotiable for the health of the transactional database under concurrent multi-tenant load.

However, legitimate business requirements exist for retrieving unbounded datasets: monthly payroll exports, compliance audits, data migration extracts, and reporting pipelines. These consumers need the complete matched set, not a single page. Forcing them to manually paginate through `/api/entries` in a loop — reconstructing cursors, handling partial failures, and assembling output — pushes complexity onto the consumer without providing any additional database safety beyond what the system already enforces internally.

The tension is between preserving the bounded synchronous contract and providing a legitimate, safe escape hatch for bulk extraction without relaxing the guarantees that protect the database.

## Decision

StarDust provides a dedicated `/api/exports` endpoint for unbounded data retrieval. A consumer posts a query filter to `/api/exports`. The API validates the query against the same schema rules as `/api/entries`, responds immediately with `202 Accepted` and an export Job ID, and enqueues the job for background materialization.

**The Chronicler** — an independent background PHP daemon (separate from the Watcher, Reconciler, and Liberator) — picks up the export job. It pages through the database internally using the same cursor-based pagination and two-query bounded fetch described in ADRs `0005` and `0006` — each iteration touches at most `page_size` rows — and streams the output to a file on disk (CSV or JSON). The consumer polls the job status and retrieves the completed artifact when ready.

The synchronous `/api/entries` endpoint is not modified. Its page limits, cursor requirement, and bounded execution guarantees remain unchanged. `/api/exports` is a separate contract with different latency expectations and delivery semantics.

## Consequences

**Positive:**

- The bounded synchronous read path is preserved without exception. No "trusted consumer" bypass or elevated page limits exist on `/api/entries`.
- Database safety is maintained during export materialization. The Chronicler reuses the same cursor pagination internally, so each database operation remains bounded by page size regardless of total export volume.
- Consumers get a clean, purpose-built interface for bulk extraction rather than a workaround built on top of the paginated API.
- Export jobs are observable: consumers can poll status, and operators can monitor queue depth and materialization throughput.

**Negative:**

- Export results are not available synchronously. Consumers requiring immediate bulk data access must accept the latency of background materialization.
- The Chronicler introduces an additional background process to operate and monitor alongside the Watcher, Reconciler, and Liberator.
- Disk usage for materialized export files must be managed. Large exports produce large files that require cleanup policies.
- Partial failure semantics differ from the synchronous path: a failed export job may leave a partial file on disk, requiring the consumer to retry the entire export.

**Rejected alternatives:**

- Relaxing page limits on `/api/entries` for "trusted" consumers — creates a two-tier contract that is difficult to enforce, easy to abuse, and undermines the bounded execution guarantee. A single misconfigured consumer can degrade the database for all tenants.
- HTTP streaming (`Transfer-Encoding: chunked`) from the synchronous endpoint — still holds a long-lived database cursor and connection for the duration of the stream. Connection drops mid-stream produce partial results with no recovery path, and the held connection counts against the pool for the entire duration.
- Allowing `COUNT(*)` or removing the cursor requirement for export-like queries — defeats the bounded read path. The Chronicler achieves unbounded retrieval by composing bounded operations, not by bypassing them.
