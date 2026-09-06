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
| `source`      | string                                             | One of: `watcher`, `reconciler`, `liberator`, `chronicler`, `api`, `bulk_api`, `export_api`, `registry`.   |
| `event`       | string (snake_case)                                | Closed event-name vocabulary per source — see "Event Vocabulary" below.                                    |
| `tenant_id`   | integer or null                                    | Required for any event tied to tenant-owned data; null for global daemon events.                           |
| `correlation_id` | string                                          | Per-request UUID for API events, per-cycle UUID for daemon events, per-operation UUID for `registry` events. Carried through any sub-events emitted within the same operation. |

Source-specific fields layer on top: the Reconciler adds `chunk_id`, `rows_claimed`, `rows_processed`; the Watcher adds `page_id`, `capacity_pct`; the Liberator adds `slot_assignment_id`, `sweep_cursor_id`; the API workers add `request_id`, `route`, `latency_ms`. Sub-fields are documented per event in the source's blueprint.

#### The `registry` source's operation scope, and `chunk_correlation_id` (2026-09-06)

The `correlation_id` row originally read "per-request UUID for API events, per-cycle UUID for daemon events", which left `source: registry` — synchronous, operator- or consumer-initiated, owned by no request and no polling loop — undefined. Nine emit sites drifted in that gap. The scope is now stated: **a `registry` event's correlation id is per-operation**, minted at the boundary of the operation a person asked for.

Three consequences follow, and the third is not obvious:

- **A sub-event inherits, it does not mint.** `page_provisioned` is emitted from inside the Watcher's provisioning window and carries the *cycle* id; `slot_reserved` carries whatever operation reserved the slot — a retype's id, a compaction's, or the claiming chunk's on the ADR `0007` exhaustion path. The advisories inherit too: `cardinality_sampled` and `spread_sampled` carry the Watcher cycle id on the periodic sweep, and the *promotion's* id when fired as ADR `0019` / ADR `0031` one-shots after a slot reaches `ready` — they report on the state that promotion produced, which is what makes ADR `0031`'s "the spread sample is compaction's built-in success check" literally checkable against the compaction it judges. A compaction therefore threads one id through every relocation it orders, each relocation's own reservation, and the advisories that follow each promotion — four levels deep.
- **An asynchronous lifecycle is one operation, not two.** `rename_started` and `rename_complete` describe a single rename that happens to span two processes and an arbitrary number of Reconciler ticks. They therefore share one id, carried on `backfill_checkpoints.correlation_id`. Before this the completion half rode the per-chunk id — correlated perfectly to its own `chunk_claimed` and to nothing upstream, which is precisely the failure the carried-through clause exists to prevent.
- **`chunk_correlation_id` is added as a source-specific field on the four `registry` completion events** (`rename_complete`, `promote_to_ready`, `delete_complete`, `model_delete_complete`), naming the tick that finished the drain. The name is taken from `stardust_reconciler_dlq.chunk_correlation_id`, which already means exactly this. The `reconciler`-source `chunk_claimed` / `chunk_complete` events are unchanged: their operation genuinely *is* the chunk.

**A note on why this needed a code change rather than an amendment softening the requirement.** The finding was originally recorded as "seven registry sites omit a required field". That was wrong in the engine's favour: `StdoutNdjsonLogger` synthesises a v4 UUID for any event whose context omits one, so no record has ever reached the wire without the field. The defect was never a missing value — it was a *meaningless* one, and it was invisible for exactly that reason. `tests/Smoke/Conventions/RegistryCorrelationTest` therefore asserts the stricter in-code rule (the id must be **passed**, not merely present), because the wire-level contract this ADR states cannot distinguish the two.

**And that scan is necessary but not sufficient.** A site can pass an id it minted itself and satisfy it completely — which is what both advisory samplers did, inside a retype promotion that already had one. Enforcing the carried-through clause therefore takes two tests: the scan, and a behavioural assertion that the events of one operation share a *value* (`tests/Smoke/LifecycleCorrelationTest`). A new event on this source needs both.

**Scope note.** This amendment covers `source: registry`. The `api` and `bulk_api` sources have the same defect at three sites — `entry_written` / `entry_updated` and their paired `exhaustion_fallback`, and the per-chunk events of one `bulkWrite()` — and are unfixed. Recorded as a separate work item rather than folded in here; the `api` source's search path already threads correctly, so this is three sites and not a source-wide gap.

### Event Vocabulary

Each source declares a closed list of event names in its feature blueprint. Examples:

- `watcher`: `poll_started`, `poll_complete`, `provision_started`, `provision_complete`, `provision_failed`, `lock_contention`.
- `reconciler`: `chunk_claimed`, `chunk_complete`, `chunk_partial` (some rows DLQ'd), `dlq_inserted`, `cache_miss`, `capacity_wait`, `coercion_null`, `lease_lost` (import-job abandoned-claim recovery: the prior worker self-aborts on a `worker_identity` mismatch and the re-claimer owns terminal state, mirroring the chronicler per ADR `0025`), `deadlock_retry` (payload in `blueprints/watcher_reconciler_daemons.md` §7), `lock_wait`.
- `liberator`: `sweep_started`, `sweep_chunk`, `sweep_complete`, `deadlock_retry`, `sweep_gap_flagged`.
- `chronicler`: `job_claimed`, `job_complete`, `job_failed`, `low_disk`, `artifact_oversized`, `gc_swept`.
- `api`: `request`, `pre_flight_rejected`, `bulk_accepted`, `payload_too_large`, `cache_miss` (Phase 4 schema-version cache refresh, per ADR `0015`; shares the name with the reconciler-source event below — the `source` field disambiguates), `entry_written`, `entry_updated`, `entry_deleted`, `exhaustion_fallback`, `bulk_chunk_committed`, `bulk_chunk_rolled_back`.

  Four of those — `entry_written`, `exhaustion_fallback`, `bulk_chunk_committed`, `bulk_chunk_rolled_back` — have been emitted from `src/Write/` since Phase 3 but were absent from this list until the update/delete surface landed. `EventVocabularyTest` scanned nine `src/` directories and `src/Write/` was not among them, so the omission stayed invisible while the suite was green. The scan now covers it, which is what surfaced them.
- `export_api`: `export_accepted` (Phase 7 — emitted by the engine-side export submission entry point after the per-tenant active-job cap check + INSERT commits; mirrors the `bulk_api`/`bulk_accepted` pattern for symmetric observability of the two submission surfaces).
- `registry`: `version_bump`, `low_cardinality_index`, `cardinality_sampled`, `spread_sampled`, `high_spread_model`, `compaction_planned`, `compaction_complete`, `page_provisioned`, `slot_reserved`, `retype_started`, `promote_to_ready`, `rename_started`, `rename_complete`, `model_renamed`, `delete_started`, `delete_complete`, `model_delete_started`, `model_delete_complete`.

  `model_renamed` is the model-level counterpart, and is deliberately a single past-tense event rather than a started/complete pair: a model's name is not load-bearing anywhere — identity is `stardust_models.id`, no snapshot caches the name, and nothing resolves a model by it — so the rename is one committed UPDATE with no asynchronous half to report on.

  **Registry events are field-scoped by default; a model-scoped counterpart carries the `model_` prefix.** `model_renamed` established that implicitly and it is now a rule, so the next model-level lifecycle does not have to re-argue it.

  `model_delete_started` / `model_delete_complete` were added on 2026-08-24 for the ADR `0038` model-deletion lifecycle. They are a pair rather than a single past-tense event because — unlike a model rename — a model deletion has an asynchronous half, and the most expensive one in the engine: `model_delete_started` reports the severance commit, `model_delete_complete` reports that the entries, the fields and the model row are gone. They are distinct names rather than `delete_started` / `delete_complete` plus a scope discriminator, for three reasons. `model_renamed` already answered this exact question with a distinct name rather than `rename_complete` with `scope: 'model'`. The note below states that `delete_complete` is the only event reporting a registry row removal, and sharing the name would make that false without amending it — a model deletion removes a model row, N field rows and M `entry_data` rows, which is a different claim. And a dashboard alerting on `delete_complete` volume would silently begin counting model deletions, which is the failure this vocabulary is closed to prevent. The discriminators elsewhere in this list — `cache_miss` across two sources — disambiguate an operation that genuinely *is* the same probe; these two carry different required payloads and are not.

  The model-purge work source otherwise reuses the `reconciler` chunk vocabulary with `queue: 'model_delete_purge'`, and reports per-chunk counts as `rows_scanned` / `rows_deleted` / `sync_rows_deleted` fields on `chunk_complete`, plus `fields_dropped` on the final chunk. `rows_deleted` rather than `rows_purged` because the rows are removed rather than rewritten, and an operator watching a destructive drain needs to see that distinction. `sync_rows_deleted` is not cosmetic: it counts the ADR `0018` dead-letter rows the in-transaction `stardust_sync_queue` delete prevented, and it is the only observable that would reveal that rule regressing.

  `delete_started` / `delete_complete` were added on 2026-08-24 for the ADR `0037` field-deletion lifecycle, and follow the `rename_*` pair for the same reason: a delete has a synchronous severance half and an asynchronous purge half, and an operator needs to tell "severed, still draining" from "gone". `delete_complete` is the only event in the vocabulary that reports a *registry row removal*, which is why it is distinct from `rename_complete` rather than a shared `backfill_complete`. The purge work source otherwise reuses the `reconciler` chunk vocabulary with `queue: 'delete_purge'`, and reports per-chunk counts as `rows_scanned` / `rows_purged` fields on `chunk_complete` rather than as new event names.

  `rename_started` / `rename_complete` were added for the ADR `0036` field-rename backfill. `rename_complete` is deliberately distinct from `promote_to_ready`: a rename touches no slot, so there is nothing to promote, and reusing the slot-lifecycle name would make the two indistinguishable in a dashboard. The rename work source otherwise reuses the existing `reconciler` chunk vocabulary, and reports per-row skips as a `rows_skipped` field on `chunk_complete` rather than as a new event name.

`deadlock_retry` was added to this source on 2026-08-25 for the ADR `0038` model purge, which retries errno 1205 / 1213 on a bounded budget. The name is shared with the `liberator` and `chronicler` sources deliberately — it is the same phenomenon, and the `source` field disambiguates exactly as it does for `cache_miss`. Note the model purge has **no gap path**: on budget exhaustion it rethrows with its cursor untouched rather than skipping rows, because a skipped chunk would leave `entry_data` rows with a dangling `model_id` that nothing would ever notice.

`lock_wait` was added to this source on 2026-08-27. `deadlock_retry` reports an attempt that will be retried; `lock_wait` reports the moment a work source stops retrying and hands the chunk back, and the two are deliberately separate so "recovered after contention" and "still blocked" are distinguishable without counting attempts against a configured budget. It carries `queue`, `attempts` and `errno`.

The five work sources that emit it — sync-queue, import-job, retype, rename and field-delete purge — return `TickOutcome::LOCK_WAIT` rather than rethrowing, because each rolls its chunk back whole with its cursor on a row that transaction owns, so the next tick retries identical work with nothing skipped and nothing failed. The model purge above keeps its rethrow for the reason recorded there. `lock_wait` is therefore not an error: it is back-pressure, and the correct alert is on its *persistence*, not its occurrence.

It is distinct from `capacity_wait` on purpose, though both mean "claimed, rolled back, backing off". `capacity_wait` means the engine is out of slot inventory and needs the Watcher to provision; `lock_wait` means another transaction held a row and will let go. Sharing a name would make sync-queue depth stop being the clean signal of filterable backfill debt that ADR `0007` designates it.

Adding a new event name requires a blueprint update. Free-form `printf`-style log lines are not permitted.

Two notes on enforcement, recorded 2026-08-24 while adding the ADR `0038` pair.

`EventVocabularyTest` scans a fixed list of `src/` directories and returns nothing for a directory it does not scan, so **where an event is emitted from decides whether the closed vocabulary is enforced at all** — the `src/Write/` omission noted above is the worked example. `src/Delete/` is scanned and `src/Schema/` is scanned by no method at all, so the model-deletion initiator and its purge work source belong in `src/Delete/` alongside their field-level siblings. A future refactor that splits them into a new package must add a scan method for it in the same commit.

The blueprint-update rule is **currently unsatisfiable for `source: registry`**, which has no blueprint under `blueprints/`. Every registry event added since `version_bump` — the spread pair, the compaction pair, `model_renamed`, the delete pair, and now the model-delete pair — was therefore added with no document to update. Recorded here rather than fixed: closing it means either writing a registry blueprint or scoping this rule to daemon sources, and either is its own change.

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
