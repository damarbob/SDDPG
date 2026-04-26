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

### Artifact Lifecycle and Retention

Because export artifacts are written to local disk on the same host as MySQL and the daemons (the zero-dependency core, ADR `0002`), unbounded artifact accumulation will starve the host filesystem and take down the entire deployment.

- **Default TTL: 24 hours from job completion.** Configurable per deployment, but the Chronicler MUST enforce a finite TTL — there is no "keep forever" mode. The TTL begins when the job transitions to `completed`, not when the consumer first downloads it.
- **Per-job size cap.** Each export job has a configurable maximum artifact size (default: 5 GB). When the streamed file reaches the cap, the Chronicler aborts the job, marks it `failed` with reason `artifact_size_exceeded`, and deletes the partial file. Consumers that need larger extracts must filter their query into multiple jobs.
- **GC sweep.** The Chronicler runs a cleanup pass on each polling cycle: any artifact whose `completed_at + ttl < now()` is deleted from disk and the job row's `artifact_path` is nulled (the job row itself is retained for audit). The same pass also deletes orphaned partial files from `failed` jobs older than 1 hour.
- **Disk-pressure circuit.** Before claiming a new export job, the Chronicler checks free disk space on the artifact directory. If free space is below 10% of the partition, it skips claiming new jobs and emits a `low_disk` event until pressure clears. Already-running jobs continue (aborting them mid-stream produces no smaller footprint).

### Per-Tenant Concurrency

Without per-tenant fairness, a single tenant queuing hundreds of large exports starves every other tenant — the noisy-neighbor failure mode the bounded read path was designed to prevent.

- **Per-tenant active-job cap: 3.** A tenant may have at most 3 export jobs in `pending` or `processing` status at any time. The 4th `POST` to `/api/exports` while three are already active returns `429 Too Many Requests` with a `Retry-After` header pointing at the oldest active job's estimated completion.
- **Round-robin job claim.** When the Chronicler picks the next pending job, it orders by `(tenant_id_round_robin_position, created_at ASC)` rather than pure FIFO across all tenants. The round-robin position is computed from `MIN(created_at)` per active tenant, ensuring that a tenant with one queued job is not blocked behind another tenant's older queued jobs.
- **No global concurrency cap from this ADR.** The total number of jobs the Chronicler processes in parallel is bounded by Chronicler worker count (deployment config), not by this ADR.

### Format Negotiation

Format selection lives in the request body, not in HTTP headers, because the artifact's format is a deferred property of the eventual file — not of the immediate `202 Accepted` response.

- **`format` field in the POST body.** Required. Accepted values for v1: `csv`, `json`. The `Accept` header on the `POST /api/exports` request applies only to the immediate `202` response payload (the job descriptor), and is independent of the artifact format.
- **Compression and delimiter options also live in the body.** Future fields (`compression: gzip|none`, `csv_delimiter`, `json_lines: bool`) extend the same body schema. The HTTP `Accept` header is never re-purposed for artifact-shape negotiation.
- **Validation happens at job submission**, not at materialization. An unsupported `format` value returns `400 Bad Request` synchronously rather than producing a `failed` job hours later.

## Consequences

**Positive:**

- The bounded synchronous read path is preserved without exception. No "trusted consumer" bypass or elevated page limits exist on `/api/entries`.
- Database safety is maintained during export materialization. The Chronicler reuses the same cursor pagination internally, so each database operation remains bounded by page size regardless of total export volume.
- Consumers get a clean, purpose-built interface for bulk extraction rather than a workaround built on top of the paginated API.
- Export jobs are observable: consumers can poll status, and operators can monitor queue depth and materialization throughput.

**Negative:**

- Export results are not available synchronously. Consumers requiring immediate bulk data access must accept the latency of background materialization.
- The Chronicler introduces an additional background process to operate and monitor alongside the Watcher, Reconciler, and Liberator.
- Disk usage for materialized export files is bounded by the per-job size cap and the artifact TTL, but not eliminated. Operators must size the artifact partition for `peak_concurrent_jobs × per_job_cap` headroom plus TTL accumulation.
- Partial failure semantics differ from the synchronous path: a failed export job leaves no consumer-visible artifact (partial files are deleted by the GC sweep), but the job row's failure reason and last-cursor diagnostic must be inspected to decide whether to retry.
- The per-tenant cap of 3 active jobs is a deliberate artificial limit, not a database protection. Operators with single-tenant deployments may raise it; multi-tenant SaaS deployments should keep it tight.

**Rejected alternatives:**

- Relaxing page limits on `/api/entries` for "trusted" consumers — creates a two-tier contract that is difficult to enforce, easy to abuse, and undermines the bounded execution guarantee. A single misconfigured consumer can degrade the database for all tenants.
- HTTP streaming (`Transfer-Encoding: chunked`) from the synchronous endpoint — still holds a long-lived database cursor and connection for the duration of the stream. Connection drops mid-stream produce partial results with no recovery path, and the held connection counts against the pool for the entire duration.
- Allowing `COUNT(*)` or removing the cursor requirement for export-like queries — defeats the bounded read path. The Chronicler achieves unbounded retrieval by composing bounded operations, not by bypassing them.
- Object-storage offload (S3, GCS, Azure Blob) for artifacts — rejected because it violates the zero-dependency core (ADR `0002`). A future ADR may introduce a pluggable artifact-sink interface, but the default deployment must function with local disk only.
- Indefinite retention with manual cleanup — rejected because the failure mode (silent disk exhaustion → MySQL undo-log write failure → multi-tenant outage) is too severe to leave to operator vigilance. A bounded TTL is mandatory.
- `Accept`-header-driven format negotiation — rejected because `Accept` describes the format of the immediate HTTP response (the job descriptor), not the deferred artifact. Coupling them creates a class of bug where the consumer's HTTP client library mutates `Accept` (e.g., adding `application/json` for the `202` body) and silently changes the artifact format.
