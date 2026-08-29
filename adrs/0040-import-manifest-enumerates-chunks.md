# 0040 - The Import Manifest Enumerates Chunks, and `rolled_back` Is Synchronous-Only

**Status:** Proposed
**Created:** 2026-08-28

## Context

[`0011`](0011-chunked-bulk-ingestion.md) §26 specifies what an import job record carries:

> **Status and manifest.** The job record carries the current status (`pending | processing | completed | failed`) and, for jobs that have produced any chunks, a manifest enumerating each chunk's outcome (`committed | rolled_back | failed`), the entity ID range, and any failure reason.

The **synchronous** path implements that shape. `BulkIngestor::ingest()` returns a `BulkIngestResult` holding a `BulkChunkResult` per chunk — `chunkIndex`, `chunkSize`, `outcome`, `entryIds`, `failureReason` — and the DTO cites §26 by name.

The **asynchronous** path, which is the one §26 is actually about, wrote `{chunks: int, entries_written: int}`. That is a resume checkpoint: `ImportJobWorkSource` reads `entries_written` back as the offset a re-claiming worker restarts from. It is not a manifest. Nothing recorded which entries landed, and a failed job reported only how far it got.

**This is not an ADR that the code outgrew.** It is one contract implemented in one mode and not the other, and §25 forces consumers across the boundary: a batch over 1,000 entities makes the synchronous call throw `payload_too_large`, so the caller must use the async path, at which point their per-chunk report silently degrades to two integers. §45 names precisely this as the surface that must be "documented and versioned together":

> Two ingestion modes (synchronous and asynchronous) must be documented and versioned together. Callers integrating bulk import must understand the size threshold, the oversized-payload exception, and the polling contract — a heavier surface than a single mode.

Separately, §26's outcome vocabulary does not fit the async path. `rolled_back` presupposes that a failed chunk is skipped and the batch continues, which is what `BulkIngestor` does. `ImportJobWorkSource` does not: the first failing chunk is terminal for the job (`failed_reason='entry_write_failed'`), which is what §38 and §42 describe — partial progress preserved, with the manifest reporting the boundary. So an async manifest can never contain a `rolled_back` record, and the reason has never been written down.

## Decision

**1. The async manifest enumerates chunks.** `stardust_import_jobs.manifest` gains a `chunk_manifest` array alongside the existing counters:

```json
{
  "chunks": 3,
  "entries_written": 1200,
  "chunk_manifest": [
    {"index": 1, "size": 500, "outcome": "committed", "entry_id_first": 1001, "entry_id_last": 1500},
    {"index": 2, "size": 500, "outcome": "committed", "entry_id_first": 1501, "entry_id_last": 2000},
    {"index": 3, "size": 200, "outcome": "failed", "entry_id_first": null, "entry_id_last": null,
     "failure_reason": "entry_write_failed"}
  ]
}
```

`chunks` and `entries_written` keep their meaning and position. They are the resume checkpoint, read arithmetically by the abandoned-claim path, and this ADR does not touch them.

**2. A range, not a list of ids.** §26 asks for "the entity ID range", and that is what is stored. The manifest is re-encoded into the job row inside *every* chunk transaction, so a full id list would put a payload proportional to the whole import inside the transaction ADR 0011 exists to keep short. `BulkIngestor` already logs `entry_id_first` / `entry_id_last` on its own chunk event; this is the same pair, made durable.

**3. `outcome` is `committed | failed` on this path. `rolled_back` is synchronous-only.** It is not a missing feature and must not be "fixed" by making the async path continue past a failed chunk — §38's partial-progress-plus-boundary is the designed behaviour, and a consumer replays from `entries_written`.

**4. At most one `failed` record, and it never moves the boundary.** The failing chunk's transaction has already rolled back when the record is appended, so it owns no `entry_data` rows and its id range is null. `chunks` and `entries_written` are left exactly as the last committed chunk set them.

**5. A job that fails before producing a chunk gets no manifest at all.** §26 scopes the manifest to "jobs that have produced any chunks". A `malformed_json` failure trips while reading the artifact, before the first window opens, so the column stays NULL. This preserves a distinction the read side already documents as load-bearing: `getImportJob()` reports `null` ("nothing committed"), never `0` ("ran, wrote nothing"). The same rule applies when the *first* chunk is the one that fails — the counters are omitted from the manifest rather than written as zeros, and only the `failed` record is present.

**6. `chunk_manifest` is optional on read.** Every job in flight across the upgrade carries a counters-only manifest, and every one of them must resume correctly. A missing or malformed `chunk_manifest` decodes to an empty list; the counters are what the resume depends on and they are untouched.

## Consequences

**Positive:**

- The two ingestion modes report the same thing. A consumer crossing §25's 1,000-entity threshold no longer loses per-chunk visibility as a side effect of batch size.
- A failed import is diagnosable. The operator sees which chunk failed, how large it was, and the exact `entry_data` id ranges that are durable — previously only a total count.
- The records survive a re-claim. An abandoned job's manifest is appended to, not restarted, so it is no less complete than an uninterrupted one's.
- No DDL. `manifest` is already a `JSON` column, so no `Bootstrapper` probe, no new table, and no change to the claim or lease protocol.

**Negative:**

- **The manifest is rewritten once per chunk, so bytes written grow as the square of the chunk count.** Measured on MySQL 8.0.13 (2026-08-28), one entry per chunk, driving the real work source:

  | chunks | manifest bytes | bytes/record | total rewritten |
  | -----: | -------------: | -----------: | --------------: |
  |     50 |          4,680 |         93.6 |          114 KB |
  |    100 |          9,453 |         94.5 |          462 KB |
  |    200 |         19,151 |         95.8 |          1.9 MB |
  |    400 |         38,351 |         95.9 |          7.5 MB |

  At **~95 bytes per record** and the default `reconcilerChunkSize` of 500, a 100,000-entity import is 200 chunks — a 19 KB final manifest and ~1.9 MB rewritten across the whole import, against the 100,000 `entry_data` rows those transactions are already writing. A 1,000,000-entity import is 2,000 chunks: a 190 KB manifest, ~190 MB rewritten, still ~95 KB per transaction. The cost becomes material somewhere in the low tens of thousands of chunks — around 10,000,000 entities at the default chunk size, where the manifest reaches ~2 MB and the rewrite total passes 19 GB.

  If a deployment reaches that scale, the fix is to move the records to a `stardust_import_chunks` table with one INSERT per chunk (O(1) per chunk instead of O(n)), which is a new ADR and a `Bootstrapper` probe. It was deliberately not built now: it costs a table, an entry in every test allowlist, and a second read in `getImportJob()`, for a cost that does not bite at any import size the engine has been asked to handle.

- The lease-loss detector's justification needs care when read. The checkpoint UPDATE carries `WHERE id = ? AND worker_identity = self`, and `rowCount() === 0` is taken to mean "a re-claimer overwrote our identity, never a no-op update". That inference rests on `entries_written` being monotonic, which is true by construction. The appended record also makes the row differ, but it is not what the guarantee rests on and must not be relied on for it.
- `ImportJob` grows a field. It is an appended constructor parameter with a default, so existing construction sites are unaffected.

**Rejected alternatives:**

- **Amend §26 to describe the shipped checkpoint.** Cheapest, and it was the framing the roadmap entry started from — that the ADR was the older artifact and the code had moved on. It does not survive reading §24 next to §26: the synchronous path already implements §26 faithfully, so an amendment would have to write down that the two ingestion modes report differently, which is the sentence §45 exists to prevent.
- **Store the full entry id list per chunk, matching `BulkChunkResult` exactly.** ~4 KB per chunk instead of ~95 bytes, re-encoded every chunk. It would make the manifest the dominant write in the chunk transaction at a few hundred chunks. §26 says "range".
- **Write the full manifest only at the terminal transition, keeping the in-transaction checkpoint compact.** Removes the rewrite cost entirely, but the records for chunks committed by a *previous* worker exist only in that worker's memory, so an abandoned job loses exactly the detail that matters most.
- **A `stardust_import_chunks` table now.** The right answer at a scale no deployment has reached; see above.

## Related

- ADR [`0011`](0011-chunked-bulk-ingestion.md) — the manifest contract this implements, and whose `rolled_back` outcome this scopes to the synchronous path.
- ADR [`0028`](0028-single-document-json-for-import-artifacts.md) — the artifact format whose decode failure is the one case that produces no manifest.
- ADR [`0025`](0025-chronicler-failure-semantics.md) — the lease/heartbeat self-abort the checkpoint UPDATE doubles as.
