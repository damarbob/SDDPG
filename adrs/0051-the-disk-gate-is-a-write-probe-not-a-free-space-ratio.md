# 0051. The Disk Gate Is A Write Probe, Not A Free-Space Ratio

**Status:** Proposed
**Created:** 2026-09-16

## Context

The Chronicler's pre-claim disk gate ([`chronicler_daemon.md`](../blueprints/chronicler_daemon.md) §2) decides, once per tick, whether to claim another export job. It answered that question with `disk_free_space() / disk_total_space()` against the artifact directory, tripping below `chroniclerLowDiskThresholdPct` (0.10) with a `low_disk` warning.

The internal build-sequencing notes carried a standing suspicion: shared hosting enforces per-account quotas above the filesystem layer, so the gate could read plenty-free while every write fails. That item explicitly recorded itself as **reasoning, not measurement**, and required verification on a quota'd host before any code — because a host that enforces quota via a loopback filesystem would report correctly and there would be nothing to fix.

### What was measured

Four loop-mounted arms on kernel `6.18.33.2-microsoft-standard-WSL2`, PHP 8.2.33, each with an 8 MiB limit on a 512 MiB filesystem and ~7 MiB of ballast, probed as the limited uid. Full records retained outside the repo.

**Rig validity gate — arm C (16 MiB ext4, no quota, 12 MiB ballast):** reported `free_pct 0.0443`, gate trips, `fwrite()` fails `errno=28 ENOSPC`. The rig detects what it is supposed to detect, so the other arms are trustworthy.

| Arm | Mechanism | `disk_total_space()` | `free_pct` | Gate trips at 0.10? | `fwrite()` |
| :-- | :-- | :-- | :-- | :-- | :-- |
| **A** | ext4 `usrquota` (per-uid) | `510873600` — **the whole filesystem** | **0.912** | **no** | fails, `errno=122 EDQUOT` |
| **D** | ext4 `prjquota` (project) | `8388608` — the project limit | 0.1245 | no | fails, `errno=122 EDQUOT` |
| **B** | XFS `pquota` (project) | `8388608` — the project limit | 0.125 | no | fails, `errno=28 ENOSPC` |
| **C** | ext4, no quota | `14410752` — the filesystem | 0.0443 | **yes** | fails, `errno=28 ENOSPC` |

**The suspicion holds, and arm A is why.** With `repquota` showing the uid at `7172 / 8192 KiB`, `disk_free_space()` reported the filesystem **91.2% free** while every write failed. No threshold on that number can detect that wall, because the number does not move.

**But the split is not the one that was predicted.** The working hypothesis was that only XFS special-cases `statvfs` for project quota and every other mechanism is blind. That is false: **ext4 project quota scopes `statvfs` too** (arm D reported exactly the 8 MiB project limit, not the 512 MiB filesystem). The real discriminator is **user quota vs project quota**, not the filesystem. Only per-uid user quota is invisible.

This matters for the deployment tier the gate exists to protect: cPanel-style hosting assigns each account a Unix uid and enforces *user* quotas — the blind case. Arms B and D, where the ratio genuinely works, are why the ratio check is retained rather than replaced.

### A second question the measurement closed

`CsvArtifactStream`/`JsonArtifactStream` detect disk-full only from a short or `false` `fwrite()`; both **discard** the return values of `@fflush()` and `@fclose()`. Had `EDQUOT` been deferred past `write(2)`, a job could have committed `artifact_bytes` for bytes the filesystem never took, reached `completed` with a truncated artifact, emitted nothing, and poisoned the ADR [`0047`](0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md) resume anchor.

It is not deferred. Across all 16 runs (4 arms × {one 4 MiB write, 4096-byte chunks} × {PHP write buffer off, default}): `short_write: true`, `fflush: true`, `fsync: true`, `fclose: true`, `silently_short: false` in every case. `EDQUOT`/`ENOSPC` surfaces at `write(2)`, which the existing check already catches. **The mid-write path needs no change**, and this ADR does not make one. The discarded flush/close returns remain unmeasured for network filesystems (NFS, which the engine's own deployment documentation already warns about for `flock`); that is a known unmeasured edge, not a defect this ADR is entitled to fix on speculation.

## Decision

### Commitment 1 — the gate's question changes

From *"how full is the partition?"* to *"can I write N bytes into `artifactDir` right now?"*. The ratio check is retained as a cheap pre-filter, not as the authority.

`chroniclerDiskProbeBytes` is a **detection-sensitivity parameter, not a reserve budget**. It asks whether that much can be written right now; it is deliberately **not** derived from `chroniclerPageSize` or from expected artifact size, because rows are variable-width and a derived number would be false precision. A passing probe proves the account can take 64 KiB, never that it can take a 5 GB export — that remains the mid-write path's job.

### Commitment 2 — the two checks are ordered and short-circuiting

The ratio runs first. **A ratio trip returns immediately and the write probe never runs.** Two independent justifications: `cause` becomes a clean partition of the tripping population (`write_probe` means the ratio passed *and* the probe failed), so there is no precedence rule and no `both` value; and a nearly-full partition is precisely when an extra write is least welcome.

### Commitment 3 — the write probe fails CLOSED at every stage

Including permission errors. This is a deliberate inversion of the ratio check's fail-open, and the reason is not symmetry but the measured alternative.

`ExportJobProcessor::process()` builds its `ArtifactStream` **outside every `try` block**, and `ArtifactStreamFactory` throws a bare `RuntimeException` on an unwritable directory. So an unwritable `artifactDir` today: claims the job, flips the row to `processing`, then throws through `tickRound()` and kills the process; the lease expires and the next worker dies identically. Under `bin/stardust tick --exports` it also ends the whole combined run, since ADR [`0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md) makes one run one failure domain and the Chronicler runs last — so every cron firing completes one round of registry work and exits nonzero.

The real comparison is therefore **a stalled queue with a per-tick `low_disk` warning** against **a claim-crash-restart loop that also truncates registry maintenance**. Both halves are pinned by `CombinedTickTest`, the second as an explicit counterfactual so the first cannot pass vacuously.

### Commitment 4 — the knob, and what `0` means

`Config::$chroniclerDiskProbeBytes`, appended last to the constructor per the project's append-only rule, defaulting to **65 536**. Chosen as "more than one filesystem block, enough to cross a block-group boundary, cheap enough to run every tick" — at the 10 s default idle interval that is one create/write/unlink per worker per 10 s, and `CombinedTick`'s round loop already exits on an idle round, so a quiet cron firing probes once.

**`0` is a complete opt-out** restoring the exact pre-0051 ratio-only, fail-open gate — *including taking no side effect on the filesystem*, since with the probe disabled the gate does not create the artifact directory either.

### Commitment 5 — one probe per tick, exposed as an immutable reading

`DiskPressureGate::sample(): DiskPressureReading` replaces the previous four public methods. This closes two defects that predate the quota question:

1. The ratio probe fell back to `sys_get_temp_dir()` when `artifactDir` did not exist, while `partition()` still reported `artifactDir` — so a `low_disk` event could name a directory that was never the one measured. The probe now always targets `artifactDir`; a missing directory yields `null` (fail open), same as any other probe failure.
2. `shouldSkipClaim()` and `freePct()` each re-probed the OS, so the `free_pct` a tick logged could be a different syscall result from the one that decided to skip.

The "each tick re-probes so a transient spike does not stick" property is **preserved and strengthened**: the tick now makes exactly one probe and reads it repeatedly, rather than two that could disagree. The reading is never stored on the gate.

### Commitment 6 — `low_disk` keeps its name and gains a closed `cause`

Payload gains `cause` (`free_pct` | `write_probe`), `probe_bytes`, `probe_stage` (`mkdir` | `open` | `write` | `flush` | `close`), and `probe_error`. `probe_stage`/`probe_error` are null when `cause` is `free_pct`, by construction of the short-circuit.

This follows the project's own rule, read off this blueprint: **distinct events for distinct *outcomes*; a closed sub-taxonomy for distinct *causes* of one outcome.** `job_failed` already carries four unrelated infrastructure causes under one name (§6 Notes: "the event field is finer-grained than the row state"), while `artifact_oversized` and `lease_lost` earned their own names because their *outcome* differs — different row state, different ownership of terminal state. A ratio trip and a probe trip produce the identical outcome: skip the claim, run GC, return `IDLE`, emit one warning.

**This does widen what `low_disk` means** — from "the partition is low on space" to "the Chronicler cannot commit to writing an artifact right now, and therefore declined to claim". That is a semantic change to an existing closed-vocabulary name and is recorded here rather than smuggled in under a sub-field. `event=low_disk AND cause=write_probe AND probe_stage=open` remains a precise dashboard selector.

**ADR [`0020`](0020-structured-logging-mandate.md) is NOT amended** — no new event name is introduced, and `EventVocabularyTest` is unchanged. Promoting `cause=write_probe` to its own event later is additive and cheap; retiring a shipped event name is not, which is why this starts with `cause`.

### Commitment 7 — the directory idiom is extracted

`Support\ArtifactDirectory::ensure(): bool` becomes the one definition of the race-tolerant `is_dir() || @mkdir() || is_dir()` idiom, replacing inline copies in `ArtifactStreamFactory` and `BulkIngestSubmitter` and serving the gate as a third caller. It lives in `Support/` rather than `Chronicler/` because it crosses two packages, on the `RetryableLockFailure` precedent.

**It returns `bool` and does not throw**, because what a caller does with `false` is caller policy — the same reason `RetryableLockFailure` declines to standardise what happens after a match. The two existing callers keep their own exception messages verbatim; the gate turns `false` into `probe_stage: 'mkdir'` and never throws.

### Commitment 8 — a leaked probe is swept by GC

`GcSweeper` gains a third bucket over `DiskPressureGate::probeGlob()`, reusing the existing `chroniclerOrphanedPartialTtlSeconds` — already exactly the "a crash stranded a file" bucket, so no fourth TTL knob. The age check is also what stops it deleting another worker's *live* probe mid-tick. Counted as `probes_deleted`, deliberately **not** folded into `artifacts_deleted`, which is normative in §6 and means artifacts.

**The per-probe-unique filename is load-bearing for multi-worker safety**, not stylistic: two `chronicler` processes share one `artifactDir`, and with a fixed name one worker's cleanup unlink deletes another's in-flight probe — on POSIX the victim then writes to an unlinked inode and the probe *passes wrongly*.

### Commitment 9 — `--exports` stays opt-in; only its reason changes

The quota gap was never its only justification, and the survivors are permanent rather than pending: ADR 0050's own Consequences note that GC is idle-cycle-only, so a Chronicler saturated with yielding exports never runs its TTL sweep — true independently of quota. Beyond that, a 64 KiB probe proves 64 KiB, not 5 GB; and flipping the default would silently start writing potentially multi-gigabyte artifacts from every existing `tick` cron line, which is exactly the class of change the project's compatibility discipline exists to prevent. Off→on later is one line; on→off after operators have built crontabs around it is breaking.

## Consequences

- The gate now takes a filesystem side effect (create, write, unlink) on every tick where the ratio passes. Bounded at 64 KiB per worker per idle interval.
- `low_disk` becomes reachable for a cause that is not disk fullness at all — a permissions misconfiguration now surfaces as a repeating `low_disk{probe_stage:mkdir}` warning with a stalled queue. That is the intended trade against a crash loop, but operators alerting on `low_disk` should route on `cause`.
- A quota with more headroom than `probeBytes` still passes the gate and fails mid-write with `failed:disk_full`. The probe narrows the window; it does not close it, and cannot.
- The blind case is now known to be *user* quota specifically. An operator on project-quota hosting was never exposed and still isn't.

## Rejected Alternatives

- **A second free-space threshold.** The original sketch of this fix. Rejected by the measurement itself: a threshold on a number reporting 91.2% cannot detect a wall one megabyte away, however it is tuned.
- **A distinct new event name for the probe trip.** Rejected as the wrong first move — the outcome is identical to a ratio trip, so it would break the distinct-outcomes rule it claims to follow. Additive and cheap later if dashboards want it.
- **Fail-open on permission errors**, on the theory that a misconfiguration should be loud. Rejected: it is already loud in the worst possible way (a crash loop), and `low_disk` with a `probe_stage` is louder in the way that helps.
- **Caching a probe result across ticks.** Rejected — the gate's recovery property is that a transient spike does not stick.
- **Probing with `ftruncate()`** instead of real bytes. Rejected: sparse files allocate no blocks and incur no quota charge, so the probe would pass under the exact condition it exists to detect.
- **Deriving the probe size from `chroniclerPageSize`.** Rejected as false precision; see Commitment 1.
- **Calling `fsync()` in the probe.** Rejected on the measurement: `EDQUOT` surfaces at `write(2)` on every arm, so `fsync` would buy nothing and cost a real disk sync every tick forever. `fflush()` is checked because it is free.

## Related

- Amends ADR [`0025`](0025-chronicler-failure-semantics.md)'s configuration-knobs table and its framing of the circuit as "<10% free".
- Amends ADR [`0050`](0050-chronicler-cooperative-yield-at-a-chunk-boundary.md) Commitment 6 and its "compose unconditionally" rejected alternative, both of which rest on this gap being unverified.
- Amends ADR [`0048`](0048-bounded-combined-tick-for-cron-driven-hosting.md)'s 2026-09-16 amendment block, which repeats the same rationale.
- Does **not** amend ADR [`0020`](0020-structured-logging-mandate.md) — no new event name.
- Leaves ADR [`0047`](0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md) untouched; the measurement confirmed the resume anchor is not exposed to a deferred-`EDQUOT` truncation.
