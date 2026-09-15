# 0047 - The Export Resume Anchor Is The Artifact Plus Its Byte Offset

**Status:** Proposed
**Created:** 2026-09-14

## Context

ADR [`0025`](0025-chronicler-failure-semantics.md) Commitment 1 describes worker-death recovery this way: a re-claimer of an abandoned `stardust_export_jobs` row "resumes from `last_cursor` with the partial artifact deleted," and asserts that "nullification of the partial file plus a fresh append from `last_cursor` produces an artifact equivalent to one written by an uninterrupted worker."

That claim does not hold, and the shipped implementation faithfully carries the error through four independently defensible pieces:

1. `ExportJobClaimer::claimAbandoned()` best-effort `@unlink`s the prior partial artifact as part of the claim.
2. `ExportJobProcessor::process()` opened with `$cursor = $job->lastCursor ?? 0` unconditionally — trusting the stored cursor regardless of how the job was claimed.
3. Both artifact streams (`CsvArtifactStream`, `JsonArtifactStream`) open their file with `fopen($path, 'wb')`, which truncates.
4. `ClaimKind` (`Pending` | `Abandoned`) was carried on every `ClaimedJob` but never read by the processor — the two claim kinds took an identical path.

The composite: the file holding rows `1..N` is deleted, and the walk restarts at row `N+1`. A consumer who downloads the "completed" artifact receives a file silently missing every row the dead worker had written. This is invisible to any test that drives one job start-to-finish without interruption — which is every test the Phase 7 suite had until this ADR, including `ChroniclerMultiWorkerClaimTest` (which proves the *claim* routes correctly, not what the *processor* does with what it claims).

A first, narrower fix (see Related) makes an abandoned re-claim always restart from row 0 — matching what `claimAbandoned()`'s unlink already does to the file, so the artifact is complete but the dead worker's progress is thrown away on every re-claim. That is safe but wasteful for the multi-GB / multi-million-row jobs the Chronicler exists to serve, and it is also a precondition the shared-hosting cooperative-yield item in the roadmap needs solved correctly before it can land — a yield that requeues a job mid-export on every tick boundary would otherwise manufacture this exact data-loss case on a schedule rather than by accident.

## Decision

**The resume anchor is not `last_cursor` alone — it is `last_cursor` bound to the specific bytes of the specific artifact file that cursor describes.** A worker may resume a prior attempt's progress exactly when it has verified it re-opened those bytes; otherwise it discards the cursor and the file together and starts over. Nothing may delete the file while keeping the cursor usable, and nothing may keep the cursor usable once the file is gone, truncated, or unidentifiable.

**Commitment 1 (supersedes ADR 0025 Commitment 1's resume half).** `stardust_export_jobs` gains a nullable `artifact_bytes BIGINT` column. Every chunk commit — not only the final one — now writes `artifact_path` and `artifact_bytes` alongside `last_cursor`, so an in-flight job's row always names the file its progress lives in and how many bytes of it are committed. `ExportJobClaimer::claimAbandoned()` stops unlinking the prior partial; it passes the path and byte count forward on the `ClaimedJob` instead.

**Commitment 2 — resume is a verified re-open, not a trusted claim.** The artifact streams open in `'c+b'` mode (create, read/write, no truncate) rather than `'wb'`, and validate a nonzero adopted anchor before trusting it: the file must exist, its size must be at least the recorded `artifact_bytes`, and — CSV only — its first line must equal the freshly-resolved header (catching a field renamed between the two attempts, since `stardust_fields.name` flips immediately per ADR [`0036`](0036-entry-payload-keys-are-field-names.md) while the file on disk was written under the old name) — JSON checks only that the file begins with `[`. An exclusive, non-blocking `flock` is acquired for the stream's lifetime; failing to acquire it is itself a resume failure. On success the stream `ftruncate`s to exactly the recorded byte count (discarding any bytes from a chunk that started but never committed), seeks there, and skips re-writing the prelude. On any validation failure it truncates to zero and starts fresh, exactly as before this ADR.

**Commitment 3 — the processor trusts the stream's verdict, not the claim kind.** `ExportJobProcessor::process()` reads `$stream->resumedFromByte()` after `open()`: nonzero means the file was genuinely adopted, and only then does the processor probe from `$job->lastCursor`; zero means start the probe at row 0, regardless of what `last_cursor` said. This makes the invariant self-enforcing at the one place that matters — a pending claim's `ClaimedJob` carries no artifact path at all (there is no prior attempt to adopt), so it always resumes at byte 0 and therefore always probes at row 0, which is what already made the pending path correct.

**Commitment 4 — the anchor dies with the file, always.** The four existing terminal-failure paths (`failDiskFull`, `failExcessiveSkips`, `failQueryFailure`, `failArtifactOversized`) already delete the partial before marking the row `failed`; `markFailed()` now also NULLs `artifact_path` and `artifact_bytes` in the same UPDATE, so a `failed` row never advertises an anchor to a file that no longer exists. Lease loss (Commitment 2 of ADR 0025) changes in the opposite direction: the losing worker used to delete its partial; it now only releases its `flock` and closes the handle, leaving the file for the re-claimer that now owns the row. Under a shared, appendable resume path, a zombie worker's delete would destroy bytes the live re-claimer is actively resuming from — the single most dangerous interaction this ADR introduces, and the reason lease-loss behavior had to change alongside the resume path rather than after it.

**Commitment 5 (amends ADR 0025 Commitment 5's accounting, does not change the cap).** `skip_count` continues to carry forward across a re-claim unchanged. On a genuine resume this is exactly right — the dead worker's rows are not re-read, so nothing is double-charged. On a fallback restart-from-zero (any of Commitment 2's validation failures), the same rows can be re-probed and re-charged, so a job that had already accumulated skips before the fallback can reach `skip_count_cap` sooner than a single accounting of the dataset would predict. This is accepted rather than special-cased: it fails closed (a loud `failed:excessive_skips` rather than a quiet undercount), and it only degrades the specific case ADR 0025 already treats as exceptional.

### New event

`artifact_resumed` (`source: 'chronicler'`, `info`): `job_id`, `worker_identity`, `resumed_from_byte`, `last_cursor`, `restart_cause`. `restart_cause` is `null` on a genuine resume, else one of `no_anchor` (pending claim, or an abandoned claim whose row carried no path — e.g. claimed before this column existed) | `missing` | `short` | `header_mismatch` | `locked`.

## Consequences

**Positive:**

- Closes the data-loss path completely rather than only the correctness half: a crashed worker's committed work up to its last chunk boundary survives a re-claim.
- The invariant is enforced structurally (the stream reports what it actually did) rather than by two call sites agreeing to stay in sync, which is exactly the class of drift that produced the original defect.
- `artifact_path` being populated on every chunk, not only the final one, incidentally makes a mid-flight crash's artifact discoverable by `GcSweeper`'s orphan bucket for the first time — previously such a partial had no path recorded anywhere the sweep could find.
- No new daemon knob and no `Config` field: the behavior is unconditional, so there is nothing to default or document as tunable.

**Negative:**

- One more schema column (`artifact_bytes`), one more `Bootstrapper::ensureXxx()` probe, and a hair more per-chunk-commit write cost (two more bound columns per `UPDATE`).
- The `flock` step adds a filesystem-locking dependency the Chronicler did not previously have. It is best-effort in the sense that a lock failure is treated as an ordinary resume-validation failure (fall back to restart-from-zero) rather than a hard error, but a filesystem that does not support `flock` reliably (some network filesystems) degrades every re-claim to a full restart — acceptable, since that is the ADR 0025-era behavior anyway, but worth stating plainly for the shared-hosting deployment item, which already flags NFS `flock` unreliability for `PidFileGuard` and now inherits the same caveat here.
- The CSV header-comparison check means a field rename that completes *between* a worker's death and its re-claim silently costs that job a full restart rather than a partial resume — correct (a stale header would otherwise leak into the artifact per ADR 0036's existing alias handling, which does not apply mid-file), but another concrete case where "resume" quietly degrades to "restart."
- `skip_count`'s conservative double-counting on a fallback restart (Commitment 5) can fail a large job that would have completed under a single honest accounting. This is accepted, not fixed, per Commitment 5's own reasoning.

**Rejected alternatives:**

- **Content-hash the resumed bytes instead of a header/prefix check.** Rejected as disproportionate: hashing a multi-GB partial on every re-claim defeats the purpose of resuming to save work, and the header/prefix check already catches the concrete case that actually changes mid-file (a rename), not an arbitrary corruption an operator would need to hash-verify against.
- **Store a per-chunk manifest (mirroring ADR [`0040`](0040-import-manifest-enumerates-chunks.md)'s import-side chunk enumeration) instead of a single byte offset.** Rejected: the import path needs per-chunk entity-id ranges because a failed *synchronous* chunk there is terminal and an operator inspects it; the export path's chunks are homogeneous and the only fact a resume needs is "how many bytes are already correct," which one integer answers. A manifest would be strictly more bookkeeping for no behavior this ADR needs.
- **Keep the delete-and-restart behavior from the interim fix and never build append-resume.** Considered and rejected as the permanent answer, not merely deferred: it is correct but throws away arbitrarily large amounts of completed work on every re-claim, and the shared-hosting cooperative-yield roadmap item cannot land safely on top of it (see Context).

## Related

- ADR [`0025`](0025-chronicler-failure-semantics.md) — Failure semantics this ADR amends. **Extended 2026-09-14 by this ADR:** Commitment 1's resume-from-`last_cursor` claim is corrected by this ADR's Commitment 1–3 (resume requires a verified re-open of the artifact, not a trusted cursor); Commitment 2's lease-loss partial-delete is corrected by this ADR's Commitment 4 (a lease-losing worker no longer deletes — it releases its lock and leaves the file to the re-claimer); Commitment 5's skip-count accounting gains the fallback-restart caveat in this ADR's Commitment 5. Every other commitment of 0025 (deadlock retry, bad-row skip, DB-disconnect backoff, disk-full, artifact-oversized) is unchanged.
- ADR [`0036`](0036-entry-payload-keys-are-field-names.md) — Why a field rename can make a resumed CSV header stale mid-file.
- ADR [`0010`](0010-asynchronous-exports.md) — The async export contract this ADR's recovery path serves.
- [`blueprints/chronicler_daemon.md`](../blueprints/chronicler_daemon.md) — Daemon blueprint; updated alongside this ADR for the re-claim/lease-loss/event-table changes.
