# 0028 - Single-Document JSON for Async Import Job Artifacts

**Status:** Proposed
**Created:** 2026-05-23

## Context

ADR `0011` established that bulk-ingest submissions above the 1 000-entity synchronous threshold MUST route through an async job: the engine persists the payload to "a job record (artifact path on local disk, identical to the export pattern)" and returns an Import Job ID immediately, leaving the Reconciler to drain the work. The decision settles _that_ an artifact exists; it does not settle the artifact's wire format. Without a declared format contract the producer (`BulkIngestSubmitter`, shipped in Phase 3) and the consumer (the Reconciler's import-job drain, scheduled for Phase 5) cannot be built independently.

Two factors raise this from an implementation detail to an architectural commitment. First, the choice binds two phases that ship months apart: the Phase 5 Reconciler must read whatever Phase 3 wrote, including artifacts produced before the Reconciler existed and queued in `stardust_import_jobs` across the version boundary. Second, the available formats trade against each other along a single axis — peak memory cost vs. implementation surface — and that trade-off is invisible to anyone reading either side of the contract in isolation.

The natural format options are a single JSON document encompassing the whole batch, newline-delimited JSON (one entity per line), or multiple smaller artifact files chunked at submission time. Each shifts the cost-vs-complexity boundary differently and the choice forecloses streaming work that would otherwise be a candidate optimisation later.

## Decision

Async import job artifacts are written as **a single JSON document encompassing the entire submitted batch**, produced by one `json_encode()` call and persisted by a single `file_put_contents($path, $json, LOCK_EX)` call. The format is not newline-delimited and is not streamed.

The document schema is:

```json
{
  "tenant_id": <int>,
  "entries": [
    { "tenant_id": <int>, "model_id": <int>, "fields": { ... } },
    ...
  ]
}
```

Files are named `import_{tenant_id}_{uuid_v4}.json` under the deployment's configured artifact directory (`Config::$artifactDir`). `json_encode` runs with `JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES`. The submitter verifies the written byte count equals `strlen($json)` and treats any short write as a fatal submission failure: the partial file is removed and no `stardust_import_jobs` row is inserted, so the Reconciler is never asked to read a truncated artifact.

The Reconciler MUST read the artifact with `file_get_contents` followed by `json_decode($contents, true, flags: JSON_THROW_ON_ERROR)`, validate the top-level shape (`tenant_id` plus an `entries` array) before any chunking, and reject the job to its DLQ on parse failure or shape mismatch. Per-chunk transaction scoping (ADR `0011`) applies _after_ the full document is loaded into memory: the Reconciler iterates the `entries` array in `BulkIngestOptions::$chunkSize`-sized windows, each window running in its own transaction. The streaming aspect of chunked ingestion is preserved at the database layer even though the artifact itself is buffered.

This decision is scoped to the async **import** path. The export path is owned by the Chronicler and remains free to choose its own artifact format (currently NDJSON per `blueprints/chronicler_daemon.md`); the two daemons do not share artifact code.

## Consequences

**Positive:**

- The artifact is a self-describing, self-validating unit. A truncated or corrupted file fails `json_decode` deterministically, so the Reconciler's DLQ rule is "the artifact either parses whole or it doesn't" — there is no partial-parse state to reason about.
- Atomic submission: `file_put_contents` with `LOCK_EX` plus the post-write size check means a reader that observes the file at any point sees either the full document or no file at all (the submitter `unlink`s on any failure before the `stardust_import_jobs` row is inserted).
- Retry and replay are trivial. An idempotency-key collision (ADR `0011`) returns the original job's id and discards the freshly written artifact; an operator-initiated retry points the Reconciler at the same path and rereads it byte-for-byte.
- Producer and consumer code are small: no incremental encoder on the write side, no streaming parser on the read side. Both sides reuse PHP's `json_encode` / `json_decode` directly.

**Negative:**

- Peak memory cost on the submitter scales linearly with the serialized payload: the source `EntryPayload` array, the intermediate associative array built by `array_map`, and the produced JSON string all coexist in PHP heap during `json_encode`. A 50 000-entity submission with ~1 KB of fields per entity peaks around 150–200 MB of resident memory before the disk write completes. Deployments must size `memory_limit` against the largest expected async batch.
- The Reconciler inherits the same memory cost on the read side. `json_decode` produces an associative array that must coexist with the source string for the duration of the call; the Reconciler's chunking loop only releases memory once it has finished consuming the decoded array.
- The artifact is opaque to ad-hoc inspection at large sizes. A 200 MB single-line JSON document is not readable by `less` or `head`; operators investigating a stuck job must use `jq` or a programmatic tool to inspect it.
- The decision forecloses streaming the producer side without a format-version bump. Future work to support arbitrarily-large async imports (e.g., off-host data loaders submitting millions of entities) will require either a follow-up ADR introducing NDJSON as a second format with a discriminator in `stardust_import_jobs`, or a redesign of the submission path entirely.

**Rejected alternatives:**

- **Newline-delimited JSON (one entity per line).** Would let both the submitter and the Reconciler stream-process without holding the whole payload in memory, eliminating the `memory_limit` ceiling on async batch size. Rejected because the Reconciler's chunked-transaction model (ADR `0011`) already operates on bounded windows downstream of the artifact read; the realized benefit is the producer-side memory ceiling, not throughput. The cost is a meaningful expansion of the producer/consumer surface — an incremental NDJSON writer that handles partial-line atomic-write semantics, a line-by-line parser on the reader that must distinguish "valid empty file" from "truncated mid-line", and a weaker truncation-detection story than single-document JSON's all-or-nothing parse. The trade is acceptable only when async batches routinely exceed what `memory_limit` tolerates, which is not the case at v0.3.0 scope.
- **Chunked artifacts (one file per N entities, manifested by a header file).** Would bound peak memory at the chunk size regardless of batch size, but introduces orphan-management complexity that doesn't otherwise exist: a submission that fails partway must clean up an arbitrary number of partial chunk files, and the `stardust_import_jobs` row would have to track a directory plus a manifest rather than a single path. Rejected as premature — the operational complexity is real and present, the memory benefit is hypothetical until batch sizes outgrow `memory_limit`.
- **Direct payload storage in `stardust_import_jobs` (no on-disk artifact).** Would eliminate the artifact-directory dependency and the disk-write atomicity question. Rejected because async submissions can exceed MySQL's `max_allowed_packet` (default 64 MB) and because the export path (ADR `0010`) already establishes the artifact-on-disk convention for large async payloads; deviating from it for imports loses an architectural symmetry without offsetting benefit. A 1 MB payload would be fine in a `LONGBLOB`; the format must serve the upper end of the size distribution, not the lower.
- **Defer the format decision to the Reconciler's implementation phase.** Rejected because `BulkIngestSubmitter` ships in Phase 3 and is already writing artifacts that will be persisted in `stardust_import_jobs.artifact_path` rows. Any format change after Phase 3 ships requires either rewriting in-flight artifacts during the migration or maintaining a multi-format reader in the Reconciler from day one; both are heavier obligations than declaring the format now.
