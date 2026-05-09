# 0025 - Chronicler Failure Semantics

**Status:** Proposed
**Created:** 2026-05-09

## Context

ADR [`0010`](0010-asynchronous-exports.md) commits the Chronicler to a multi-worker model: claim via `SELECT ... FOR UPDATE SKIP LOCKED`, materialize the artifact, mark the job `completed`. ADR [`0010`](0010-asynchronous-exports.md) does not pin **what happens when a worker does not finish**. Five failure modes need pinned semantics before the daemon can be implemented:

1. **Worker crashes mid-job.** A row is left in `status='processing'` with no live owner. Without a recovery path, the job is stranded; an operator must manually reset it. The Liberator's "restart resumes from last checkpoint" model does not transfer — the Liberator is a singleton (one PID, restart-by-process), the Chronicler is multi-worker (recovery means *another worker takes over*).
2. **Repeated chunk-level deadlocks.** The Liberator pinned a 3-retry budget per chunk, then logs `sweep_gap_flagged` and advances ([ADR 0009](0009-tombstone-based-slot-eviction.md)). The Chronicler reads from the same paged tables under the same MySQL InnoDB regime; without a parallel budget, a contended chunk blocks an export indefinitely.
3. **Row payload incompatible with chosen format.** A JSON-incompatible byte sequence in a CSV cell, an unrepresentable Unicode codepoint in JSON output, an embedded NUL — these cannot be encoded into the artifact for that row. Aborting the entire job on one bad row is hostile to multi-GB exports; silently dropping is hostile to consumers expecting completeness. The Reconciler's `coercion_null` precedent ([ADR 0024](0024-type-coercion-matrix-for-retype-backfill.md), [`watcher_reconciler_daemons.md`](../blueprints/watcher_reconciler_daemons.md) §7) is the existing "skip + emit observable event" pattern, but the predicate and bound are not yet authored for the export path.
4. **Database connection drop mid-pagination.** A network blip between the Chronicler and MySQL during a long-running export. Treating it as a generic exception conflates infrastructure events with data-shape failures and forces operators to read stack traces to triage.
5. **Filesystem fills mid-write.** The disk-pressure circuit ([`chronicler_daemon.md`](../blueprints/chronicler_daemon.md) §2) gates *new* claims at <10% free. It does not address an in-flight job whose append exhausts the partition mid-stream. The semantics — abort, retry, requeue — are unauthored.

ADR [`0020`](0020-structured-logging-mandate.md) has already committed the Chronicler to a closed structured-log event vocabulary. The events pinned in this ADR's decision section are the events that observability dashboards and alerts will key on; once shipped they cannot be renamed without a coordinated blueprint update.

## Decision

Six commitments govern Chronicler failure semantics. The schema column additions in [`schemas/schema_reference.md`](../schemas/schema_reference.md) §5.2 (`claimed_at`, `heartbeat_at`, `skip_count`, extended `failed_reason` enum) are the persistence substrate; this ADR pins the engine semantics.

**Commitment 1 — Lease + heartbeat for worker-death recovery.** A claimed job row carries `claimed_at` (set on the `pending → processing` transition, in the same transaction as the `SKIP LOCKED` claim) and `heartbeat_at` (refreshed on every chunk-commit transaction, default 5s cadence — i.e., a chunk commit always also writes `heartbeat_at = NOW()`). A second `SELECT ... FOR UPDATE SKIP LOCKED` poll, distinct from the pending-job poll, scans `WHERE status='processing' AND heartbeat_at < NOW() - INTERVAL <lease_timeout> SECOND` (default 30s). Rows it claims are *abandoned* — the original worker is presumed dead — and the new worker resumes from `last_cursor` with the partial artifact deleted. Recovery is idempotent because nullification of the partial file plus a fresh append from `last_cursor` produces an artifact equivalent to one written by an uninterrupted worker.

**Commitment 2 — A worker that detects its own lease lapse aborts cleanly.** When a worker's chunk commit reads back its own row and observes that `worker_identity` no longer matches its identity (a re-claimer overwrote it), the worker emits `lease_lost`, releases all local file handles, deletes any partial artifact it owns, and exits the job loop without further writes. This prevents two workers writing the same artifact path concurrently. The lease-lost worker does NOT mark the row `failed`; the re-claimer is now responsible for the job's terminal state.

**Commitment 3 — Bounded deadlock retry per chunk.** On `SQLSTATE 40001` during a chunk's bounded read (Query 1 or Query 2 per [ADR 0005](0005-two-query-bounded-read-path.md)), the Chronicler retries the same chunk from the same `last_cursor` after the inter-chunk delay. After **three consecutive deadlocks** against the same chunk, the Chronicler emits `chunk_skipped` with `cause='deadlock_budget_exhausted'`, advances `last_cursor` past the contended range by `page_size`, increments `skip_count` by `page_size` (worst case — the actual skipped row count is unknown because the chunk never returned), and continues. The job still reaches `completed` if no further failures occur. Operators reviewing a `chunk_skipped` event decide whether to re-run the export. This mirrors the Liberator's `sweep_gap_flagged` policy: bound pathological contention rather than block forever.

**Commitment 4 — Bad-row skip policy.** When format-encoding a row produces a representation error — the row's value contains a byte or codepoint that cannot be expressed in the chosen `format` (`csv` | `json`) — the Chronicler skips that row, increments `skip_count` by 1, and emits `row_skipped` with `entry_id` and a closed-taxonomy `reason`. The job continues. Closed reason taxonomy: `format_invalid` (the value's bytes do not survive the chosen format's escape rules — e.g., embedded NUL in CSV, lone surrogate in JSON), `unrepresentable_codepoint` (a Unicode codepoint outside the format's expressible range — e.g., non-BMP without UTF-8 in legacy CSV consumers). Per-row independence mirrors [ADR 0024](0024-type-coercion-matrix-for-retype-backfill.md): a single bad row does not abort the chunk and does not delay export progress.

**Commitment 5 — Bounded skip budget per job.** When `skip_count` for a job exceeds `skip_count_cap` (default 1000) the Chronicler aborts the job: marks `status='failed'`, sets `failed_reason='excessive_skips'`, deletes the partial artifact, and emits `job_failed` (no `row_skipped` is emitted for the cap-triggering row itself — the abort event is sufficient). The cap exists so that a malformed slice of a tenant's data — schema drift, encoding regression — surfaces as a job failure rather than a silent multi-million-row hole in an artifact a consumer is about to depend on. The cap counts both `row_skipped` increments and the `page_size` charge from a `chunk_skipped` event.

**Commitment 6 — Infrastructure failure events are distinct from generic failure.** Three infrastructure conditions — DB disconnect, filesystem full, lease timeout — get dedicated terminal handling so operator dashboards can route them differently from data-shape failures:

- **DB disconnect mid-pagination.** Reconnect with exponential backoff: 1s, 4s, 16s (3 attempts). On exhaustion, mark `status='failed'` with `failed_reason='query_failure'`, preserve `last_cursor` for operator-initiated restart, emit `job_failed` with `reason='db_disconnect_exhausted'`. Backoff schedule is fixed in this ADR rather than a tunable: a longer backoff masks an outage worth paging on; a shorter one wastes connection-pool capacity.
- **Filesystem full mid-write.** Detect `ENOSPC` on append, mark `status='failed'` with `failed_reason='disk_full'`, delete the partial artifact (best-effort — if the delete itself fails the operator inspects via `worker_identity`), emit `job_failed` with `reason='disk_full'`. This is distinct from the disk-pressure circuit's `low_disk` event, which gates *new claims*: once a job is in flight, the disk-pressure threshold is no longer protective, and a `disk_full` mid-write is the failure surface.
- **Lease lost (self-detected).** Per Commitment 2, emit `lease_lost` and exit silently. The re-claimer's terminal event (`job_complete`, `job_failed`) carries the job's final state.

### Failure → event → status mapping

| Failure | Detection | Worker action | Event | Terminal status |
|:---|:---|:---|:---|:---|
| Crashed worker | Re-claimer's `heartbeat_at < NOW() - lease_timeout` poll | Re-claim, delete partial, resume from `last_cursor` | `job_claimed` (re-claim) | (unchanged — recovery, not failure) |
| Self-detected lease loss | Chunk commit reads `worker_identity != self` | Release handles, delete partial, exit | `lease_lost` | (unchanged — re-claimer owns terminal) |
| Deadlock budget exhausted | 3rd `SQLSTATE 40001` on same chunk | Advance `last_cursor` by `page_size`, charge `skip_count` | `chunk_skipped` | `completed` (unless skip cap also hit) |
| Bad row (format-incompatible) | Encoding error on per-row serialize | Skip row, charge `skip_count` | `row_skipped` | `completed` (unless skip cap also hit) |
| Skip cap hit | `skip_count > skip_count_cap` | Abort, delete partial | `job_failed` | `failed` (`excessive_skips`) |
| DB disconnect (after 3 retries) | Backoff schedule exhausted | Abort, preserve `last_cursor` | `job_failed` | `failed` (`query_failure`) |
| Disk full (in-flight) | `ENOSPC` on append | Abort, delete partial | `job_failed` | `failed` (`disk_full`) |
| Artifact size cap hit | Cumulative bytes written > 5 GB | Abort, delete partial | `artifact_oversized` | `failed` (`artifact_size_exceeded`) |

### Configuration knobs and defaults

| Knob | Default | Notes |
|:---|:---|:---|
| `heartbeat_cadence_seconds` | 5 | Chunk-commit cadence is the lower bound; heartbeat piggybacks. |
| `lease_timeout_seconds` | 30 | Must be ≥ 6× heartbeat cadence to absorb single-cycle worker hesitation. |
| `deadlock_retry_budget` | 3 | Mirrors Liberator. |
| `db_disconnect_backoff_seconds` | `[1, 4, 16]` | Fixed schedule; not per-deployment tunable. |
| `skip_count_cap` | 1000 | Job-scoped, not chunk-scoped. |
| `page_size` | inherits the synchronous read path | Per [ADR 0006](0006-cursor-based-pagination.md). |
| Disk-pressure threshold | 10% free | Pre-existing; gates new claims only. |
| Artifact size cap | 5 GB | Pre-existing; per [ADR 0010](0010-asynchronous-exports.md). |

## Consequences

**Positive:**

- A crashed Chronicler worker no longer strands a job. Recovery is automatic and idempotent — operators do not need a manual "reset stuck jobs" CLI in normal operation.
- Pathological MySQL contention bounds at three retries per chunk rather than blocking an export indefinitely. The `chunk_skipped` event makes the gap visible without forcing a re-run.
- Format-incompatible rows neither poison a multi-GB export (the per-row skip survives) nor silently fail (each `row_skipped` is observable, and the cumulative `skip_count` cap aborts before the loss becomes invisible).
- Three distinct infrastructure failure paths (DB drop, disk full, artifact oversized) get their own terminal event names. Dashboards can alert on `disk_full` separately from generic `job_failed`, even though both terminal states share the `status='failed'` row state.
- Heartbeat cadence (5s) is a chunk-commit free-rider — no separate timer or transaction. A worker that cannot commit a chunk also cannot heartbeat, which is the correct semantic: a worker that has lost its connection should lose its lease.

**Negative:**

- The 30s lease timeout means a worker pause longer than 30s (e.g., a stop-the-world GC, a network blip, a `kill -STOP`) gets re-claimed. The original worker, on resume, will detect lease loss via Commitment 2 and exit cleanly — no double-write — but the work it had done since its last chunk-commit is discarded. Tuning the lease timeout up reduces this risk but slows crash recovery; 30s with a 5s heartbeat is the balance point.
- The deadlock skip charges `skip_count` by `page_size` even though the actual skipped row count is unknown (the contended chunk never returned). This is conservative: it triggers the skip cap earlier than a perfect accounting would, but a job hitting the deadlock budget repeatedly is already failing in a way the operator should investigate.
- The skip cap is a single integer threshold for two distinct skip causes (per-row format failures and per-chunk deadlocks). Operators triaging an `excessive_skips` failure must consult the structured log to disambiguate. A two-cap scheme was rejected as more tunable surface than the operational value warrants.
- A re-claimer takes over a job at the boundary of a *committed* chunk. Any rows the previous worker wrote between its last chunk commit and its crash are not in the partial artifact (which the re-claimer deletes) and are re-read from MySQL. This is correct but slightly wasteful in the rare case where the previous worker had written an entire un-committed chunk seconds before crashing.

**Rejected alternatives:**

- **Operator-manual recovery only.** The `worker_identity` column already supports manual operator intervention — an operator can identify stranded jobs and reset them. Rejected as the primary recovery path because it scales poorly: even a few percent of jobs hitting the recovery path under load (a node restart during a deploy) would burn operator attention. Manual reset remains available as a fallback for cases the lease model does not cover (e.g., a worker that hangs but keeps heartbeating).
- **Lease without heartbeat (claim TTL = wall-clock job duration limit).** Setting a fixed TTL from `claimed_at` would force every job to commit within a hard wall-clock budget, which conflicts with the 5 GB / multi-million-row job sizes the Chronicler must support. Heartbeat-based liveness decouples lease length from job duration.
- **Coordination via Redis / external lock service.** A second moving part with its own failure modes, violating the zero-dependency core ([ADR 0002](0002-mysql-native-zero-dependency-core.md)). MySQL row locks via `SKIP LOCKED` plus the heartbeat column give the same liveness signal without adding infrastructure.
- **Fail the job on first deadlock.** A single deadlock against a hot partition would abort multi-GB exports, defeating the bounded-read-path's contention protection. The 3-retry budget mirrors Liberator and matches the same MySQL contention class.
- **Fail the job on first bad row.** Rejected as hostile to multi-GB exports where a single encoding artifact in a tenant's data (often itself an upstream bug the consumer is using the export to find) would block all progress. The skip + log + cap pattern preserves observability without sacrificing throughput.
- **Per-format dedicated bad-row events (`csv_invalid`, `json_invalid`).** Multiplies vocabulary without operational benefit; the `format` column on the job row plus the `reason` sub-field on `row_skipped` carry the same triage information.

## Related

- ADR [`0002`](0002-mysql-native-zero-dependency-core.md) — Zero-dependency core (rejects external lock services).
- ADR [`0005`](0005-two-query-bounded-read-path.md) — The bounded read path the Chronicler uses for chunked pagination.
- ADR [`0006`](0006-cursor-based-pagination.md) — Cursor encoding for `last_cursor`.
- ADR [`0009`](0009-tombstone-based-slot-eviction.md) — The Liberator's deadlock budget that this ADR mirrors.
- ADR [`0010`](0010-asynchronous-exports.md) — The async export contract; pins TTL, per-tenant cap, format negotiation.
- ADR [`0015`](0015-database-as-sole-daemon-coordination-point.md) — The database-only coordination model the lease/heartbeat respects.
- ADR [`0018`](0018-reconciler-poison-pill-semantics.md) — The Reconciler's per-row skip pattern that the bad-row policy parallels.
- ADR [`0020`](0020-structured-logging-mandate.md) — Event-vocabulary discipline; the events declared here extend the Chronicler list.
- ADR [`0024`](0024-type-coercion-matrix-for-retype-backfill.md) — Per-row skip + observable event precedent.
- [`blueprints/chronicler_daemon.md`](../blueprints/chronicler_daemon.md) — The daemon blueprint that pins acceptance criteria and event payloads on top of this ADR.
- [`schemas/schema_reference.md`](../schemas/schema_reference.md) §5.2 — `stardust_export_jobs` schema additions backing this ADR.
