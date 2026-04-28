# 0020 - Structured JSON Logging Mandate for Daemons and API

**Status:** Proposed
**Created:** 2026-04-23

## Context

The StarDust runtime emits diagnostic output from at least seven sources: the Watcher, the Reconciler, the Liberator, the Chronicler, the synchronous API workers, the async-job submission API workers (ADR `0011`), and the schema-registry advisory pipeline (ADR `0019`). These sources currently log to stdout in unstructured plain text — the Watcher/Reconciler blueprint flagged "structured JSON to stdout" as merely "recommended."

The architecture relies on operator vigilance for several silent-failure classes:

- **Index Desync detection** (ADR `0016`): an `unmapped_pending_promotion` field is invisible to consumers but matters to the Watcher's prioritization decisions.
- **DLQ depth/age** (ADR `0018`): poison pills accumulate without an operator response unless thresholds are alerted on.
- **Liberator stalls** (ADR `0009`): tombstoned slots silently consume capacity if the Liberator is wedged.
- **Schema-cache staleness** (ADR `0015`): the version-row probe should be sub-millisecond; a slow probe is a leading indicator of registry-table contention.
- **Coercion-failure NULLs** (ADR `0016`): an entry whose value couldn't coerce stores NULL in the new slot and gracefully falls back to JSON_EXTRACT — invisible until an operator audits backfill completion.
- **Low-cardinality index advisory** (ADR `0019`): the entire cardinality-policy mechanism produces no consumer-visible signal; it lives in the log.

Every one of these depends on machine-readable observability. Plain-text logs cannot drive alerts, dashboards, or threshold queries without ad-hoc parsing.

## Decision

All StarDust runtime processes — the four daemons, the synchronous API workers, the async-job API workers — MUST emit a structured JSON event per logical operation to stdout. Plain text logging is removed from the supported surface.

### Event Shape

Every event is a single-line JSON object terminated by `\n` (newline-delimited JSON / NDJSON). The minimum required fields are:

| Field         | Type                                               | Description                                                                                                |
| :------------ | :------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- |
| `ts`          | string (RFC 3339 UTC)                              | Event timestamp.                                                                                           |
| `level`       | string (`debug`/`info`/`warn`/`error`)             | Severity.                                                                                                  |
| `source`      | string                                             | One of: `watcher`, `reconciler`, `liberator`, `chronicler`, `api`, `bulk_api`, `registry`.                 |
| `event`       | string (snake_case)                                | Closed event-name vocabulary per source — see "Event Vocabulary" below.                                    |
| `tenant_id`   | integer or null                                    | Required for any event tied to tenant-owned data; null for global daemon events.                           |
| `correlation_id` | string                                          | Per-request UUID for API events, per-cycle UUID for daemon events. Carried through any sub-events emitted within the same operation. |

Source-specific fields layer on top: the Reconciler adds `chunk_id`, `rows_claimed`, `rows_processed`; the Watcher adds `page_id`, `capacity_pct`; the Liberator adds `slot_assignment_id`, `sweep_cursor_id`; the API workers add `request_id`, `route`, `latency_ms`. Sub-fields are documented per event in the source's blueprint.

### Event Vocabulary

Each source declares a closed list of event names in its feature blueprint. Examples:

- `watcher`: `poll_started`, `poll_complete`, `provision_started`, `provision_complete`, `provision_failed`, `lock_contention`.
- `reconciler`: `chunk_claimed`, `chunk_complete`, `chunk_partial` (some rows DLQ'd), `dlq_inserted`, `cache_miss`, `capacity_wait`.
- `liberator`: `sweep_started`, `sweep_chunk`, `sweep_complete`, `deadlock_retry`, `sweep_gap_flagged`.
- `chronicler`: `job_claimed`, `job_complete`, `job_failed`, `low_disk`, `artifact_oversized`, `gc_swept`.
- `api`: `request`, `pre_flight_rejected`, `bulk_accepted`, `payload_too_large`.
- `registry`: `version_bump`, `low_cardinality_index`, `cardinality_sampled`.

Adding a new event name requires a blueprint update. Free-form `printf`-style log lines are not permitted.

### Routing

The runtime writes JSON to stdout only. Forwarding to Loki, Vector, Datadog, Splunk, or any other destination is the deployment's responsibility — typically via a sidecar (e.g., promtail, fluentbit) or via the platform's stdout collector. The runtime never opens a network socket for logging. This preserves the zero-dependency core (ADR `0002`).

### Stderr is for crashes only

Stderr is reserved for fatal-process panic output (PHP errors, unhandled exceptions, segfaults). Operational events — even at level `error` — go to stdout as JSON. This separation lets log-shipping pipelines treat stdout as structured ingest and stderr as alertable crash-loop signal.

## Consequences

**Positive:**

- All silent-failure classes the architecture depends on operators to monitor become alertable: DLQ depth, sweep gaps, low_disk, low_cardinality_index, coercion_null counts, version_bump latency, etc.
- A single deployment-level log pipeline (whatever the operator picks) handles all StarDust runtime sources without per-source parser code. The format is uniform.
- The closed event vocabulary forces design discipline: a new operational signal requires a blueprint update, which makes it discoverable and reviewable rather than buried in logging code.
- Correlation IDs let operators trace a single request across the API → daemon boundary (e.g., a write that triggered an Exhaustion Fallback can be followed from the API event to the eventual Reconciler `chunk_complete` event).

**Negative:**

- JSON logs are less skim-friendly than plain text for interactive debugging. Operators in a tail session must use a JSON pretty-printer (`jq`, `pino-pretty`-style filter) for live inspection.
- Adding a new event name requires a blueprint edit, slowing exploratory debugging where engineers might want a one-off `printf`. The trade-off is intentional: the next instance of "we wish we had a log line for X" becomes a blueprint change rather than a bespoke parser rule.
- Disk and stdout-pipe throughput cost rises modestly versus terse plain text. JSON encoding overhead per event is small; the volume is bounded by the polling intervals and request rates.

**Rejected alternatives:**

- **Plain text with strict format ("$LEVEL $SOURCE: $msg")** — looks structured but isn't: any code path that drops a colon or interpolates a tenant-supplied string breaks downstream parsers silently. JSON's escape semantics make this category of bug impossible.
- **Direct integration with a specific log backend (e.g., Sentry, Datadog SDK)** — couples the daemon to a vendor and violates the zero-dependency core. Stdout-and-let-the-deployment-decide is the only routing model compatible with ADR `0002`.
- **Open-ended event names ("anything goes")** — defeats the discipline goal. A dashboard built against `dlq_inserted` breaks silently when a developer ships `dlq_added` six months later. The closed vocabulary makes such drift a visible blueprint change.
- **Logfmt or another non-JSON structured format** — works but every modern log pipeline parses JSON natively, while logfmt requires per-pipeline support. JSON is the broadest path of least resistance.

## Related

- ADR `0002` — MySQL Native, Zero-Dependency Core
- ADR `0009` — Tombstone-Based Slot Eviction (sweep_gap_flagged)
- ADR `0015` — Database as Sole Daemon Coordination Point (version_bump)
- ADR `0016` — Field type change and filterability-promotion lifecycle (coercion_null)
- ADR `0018` — Reconciler Poison-Pill Semantics (dlq_inserted)
- ADR `0019` — Index Cardinality Policy (low_cardinality_index)
- All blueprints under `blueprints/` — must declare their event vocabulary
