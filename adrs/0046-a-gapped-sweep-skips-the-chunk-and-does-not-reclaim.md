# 0046 - A Gapped Sweep Skips the Chunk and Does Not Reclaim

**Status:** Proposed
**Created:** 2026-09-06

## Context

ADR `0009`'s gap path exists so that a hot read partition cannot block sweep progress indefinitely: after `deadlockRetryBudget` consecutive lock failures against one chunk, the Liberator abandons it, annotates the registry row, and moves on. Two things about the implementation diverge from what `0009` says, and **both are the code drifting from the ADR rather than the ADR accepting a risk.** That distinction matters, because it means neither needs a decision reversed — only the text honoured.

### Divergence 1 — the gap advances by a span of ids, not by a chunk

ADR `0009`'s deadlock policy: *"the Liberator logs the slot identity and **skips ahead by `LIMIT` rows**"*. `SlotSweeper` implemented that as `$gapEnd = $cursor + $this->chunkSize`, which is arithmetic on `entry_id` values. `selectChunkRowIds()` walks `WHERE entry_id > ? ORDER BY entry_id LIMIT ?`, so the two coincide only where the page's ids are dense and start immediately above the cursor.

Ids in the window are strictly increasing and strictly greater than the cursor, so the k-th is at least `cursor + k`. The arithmetic therefore **under**-advances and never over-skips: nothing is passed over unswept. What is lost is progress. The next iteration re-selects the same poisoned rows, burns the whole budget again, and creeps forward one `chunkSize` of *id space* per cycle.

**Measured on MySQL 8.0.13** — four rows on a page, two low ids and two above an `AUTO_INCREMENT` jump to 200, `chunkSize = 2`, `deadlockRetryBudget = 3`, contention injected on every nullification UPDATE:

```text
before:  101 sweep_gap_flagged events, sweep_gap_count = 101
after:     2 sweep_gap_flagged events, sweep_gap_count = 2
```

Two populated chunks means two gaps. The counter that exists to tell an operator how many chunks were skipped instead counted how many times the daemon failed to get past one — a fiftyfold overstatement here, and unbounded in the id sparsity. `sweep_gap_flagged`'s `start_id` / `end_id` were wrong for the same reason: they named a cursor span rather than the chunk's range, which `liberator_daemon.md` AC#8 already specifies.

This is why `SchemaFixture::IDENTITY_TABLES` had to reset `entry_data`'s `AUTO_INCREMENT`: dropping the table between tests used to guarantee dense 1-based ids, and the suite never saw the divergence. A fixture was holding the defect still.

### Divergence 2 — a gapped sweep reclaims a slot it did not empty

ADR `0009` step 4: *"Once the Liberator confirms a slot is **100% nullified** for a given tenant/model partition, it updates the registry to mark the slot `free`. **Only then** is the slot safely available for mapping to a new field."* Its Consequences restate it: *"Data bleeding is structurally impossible. A new field can only be mapped to a slot that has been **verified empty**."*

A sweep that abandoned a chunk has confirmed no such thing. The implementation reclaimed anyway — the gap path advanced the cursor, the loop ran on to a short chunk, took `isLast`, and performed the ordinary `tombstoned → free` transition.

**Measured on 8.0.13** — twenty rows, `chunkSize = 10`, one gap, then a reservation through the real `SlotReserver`:

```text
AFTER GAP RECLAIM: status=free      gap=1  cursor=20  residue=10
AFTER RE-RESERVE:  status=assigned  gap=1  cursor=20  residue=10   (same slot)
```

Ten rows of the departed field's values handed to the next occupant. Whether they then become visible depends on who reserves next — a retype or promotion backfill rewrites the whole partition and papers over it, but the ADR `0007` exhaustion path does not, so rows written before the queue existed keep the residue and match filters they should not. This is precisely the data-bleeding failure `0009` was written to prevent.

`LiberatorDeadlockRetryTest::testThreeConsecutiveDeadlocksTriggersSweepGap` has asserted `status = 'free'` alongside ten surviving rows since Phase 6a. The two assertions together always described a bleed; nothing read them that way.

### The blueprint contradicts the ADR it cites

`liberator_daemon.md` says a gap is non-fatal and that operators may *"accept the gap (the rows in the gap range are a small fraction of one slot column, and the field's data is still authoritative in `entry_data.fields` per ADR 0013)"*. That is a sound argument that the **departing** field loses nothing. It is not an argument about whether the **arriving** field inherits something, which is the question reuse raises and the one `0009` step 4 answers. Per the project's document-precedence rule — on conflict between a blueprint and an ADR, the ADR governs — the blueprint is what is wrong here.

`glossary.md`'s *Tombstoned Slot* entry is correct as written and needs no change: it says a slot "cannot be mapped to a new field until The Liberator successfully processes and nullifies all residual data". That was true of the design and false of the code.

## Decision

**1. The gap skips the chunk.**

```php
$gapEnd = max($newCursor, $cursor + $this->chunkSize);
```

`$newCursor` is the chunk's real last id, already in scope.

**The `max()` is conservative rather than load-bearing, and the roadmap entry that proposed it overstated its role — as did the first draft of this ADR.** On a full chunk it always selects `$newCursor`, since the k-th id in the window is at least `cursor + k`. It differs only on a partial or empty chunk, which is by definition the last one, so there is nothing beyond it to over-skip. What it buys is the property that a gap never advances by *less* than the pre-0046 behaviour did.

It does **not** prevent a spin on an empty chunk, which is what it was originally justified by. Measured: under persistent contention on the registry statement an empty chunk's gap does not loop at all — `commitGap()` is invoked from inside the retry handler and rethrows, so the failure escapes `sweep()` entirely and (per ADR `0038`, `PollLoop` deliberately does not catch) the daemon exits. Under *transient* contention the sweep terminates with or without the `max()`. The expression is kept because it is conservative and free, not because it closes a hang.

**2. `sweep_gap_flagged` reports the chunk's range**, `start_id = $rowIds[0] ?? $cursor`, satisfying AC#8 as written. `sweep_gap_count` consequently counts abandoned chunks, which is what its name claims.

**3. A sweep that abandoned any chunk does not reclaim.** The final chunk commits its nullification as normal but leaves `status = 'tombstoned'`, and **no `sweep_complete` is emitted** — AC#11 defines that event as one per slot transitioned to `free`, so silence here is the contract rather than a hole in it. The operator's signal is the `sweep_gap_flagged` that already fired plus a `sweep_gap_count` that keeps climbing.

**4. It is a deferral, not a refusal.** The final chunk rewinds `sweep_cursor_id` to the cursor of the *first* gap in this pass, in the same transaction. The slot stays tombstoned, so the next Liberator cycle picks it up and re-walks from the first thing it missed — not from the beginning of the page, since everything below that cursor was genuinely swept. Once a pass completes with no gap, it reclaims normally.

**5. `sweep_gap_count` accumulates across passes and resets at the next tombstone**, per ADR `0045`. A slot under sustained contention therefore shows a climbing counter while staying tombstoned, which is exactly the "surfacing pathological contention" `0009`'s deadlock policy asks for.

No DDL, no new configuration, no new event name, no change to any event's set of fields.

## Consequences

**Positive:**

- ADR `0009` step 4 becomes true of the implementation: a slot reaching `free` has been verified empty, with no exception. That is what makes reuse safe, and it is the precondition ADR `0045` was written against.
- Sweep throughput under contention improves by the id sparsity of the page rather than marginally — the measured case went from 101 budget cycles to 2, and the gap between them grows without bound as ids get sparser.
- `sweep_gap_count` and `sweep_gap_flagged` become usable: one counts chunks, the other names the range that was actually skipped. Neither did before.
- `SchemaFixture::IDENTITY_TABLES` loses `entry_data`, saving one `ALTER TABLE` (~18 ms) per test. Nothing reads an `entry_data` id as a literal any more.
- The `sweep_gap_count > 0` slot is no longer reachable in a live status through the normal path, so the attribution ADR `0045` gave that counter becomes a diagnostic for a tombstoned slot rather than a warning about a live one.

**Negative:**

- **A slot under permanent contention never reclaims**, and its capacity stays squatted. This is the real cost, and it is a deliberate inversion of the balance `0009`'s *runtime profile* struck — that section chose bounded progress over blocking. Its *Decision* section chose correctness over both, and where the two halves of one ADR disagree the normative one wins. The justification is asymmetry: squatting is visible (tombstone depth and `sweep_gap_count` are both already monitored) and self-healing once contention subsides, whereas bleeding is silent and unrecoverable.
- **Repeated passes cost repeated work.** A slot that keeps gapping is re-swept from its first gap on every Liberator tick, so sustained contention on an early chunk means sustained re-walking of everything after it. Bounded per tick, unbounded across ticks.
- **The rewind can lose good work.** A pass that gaps on chunk 1 and then successfully sweeps chunks 2–50 rewinds to chunk 1's cursor, and the next pass re-nullifies all of it. Idempotent, but not free. Rewinding to the *first* gap rather than to zero is what keeps this proportional to where the contention was.
- A long-running gap on an early chunk now delays reclamation indefinitely rather than reclaiming with a hole, which changes what an operator must watch: tombstone depth becomes the leading indicator instead of `sweep_gap_count` alone.
- **A permanently gapping slot holds a permanent place at the head of the sweep queue, and can starve newer tombstones.** `TombstonedSlotRepository::loadBatch()` orders `tombstoned_at ASC` and `Liberator::tick()` sweeps every slot in that batch, so a slot that never reclaims keeps its original `tombstoned_at` and is re-attempted on every cycle. Where the number of stuck slots approaches `Config::$liberatorBatchSize` (default 50), tombstones behind them are never reached. Before this ADR the same slot would have reclaimed and left the queue — incorrectly, but it left. Operators with a deep tombstone backlog and persistent contention should watch queue depth as well as the individual counters; the mitigation is to relieve the contention, or to raise the batch size so the stuck head does not consume the whole cycle.

**Rejected alternatives:**

- **Re-sweep the gap range inline before reclaiming.** Turns a bounded sweep into a potentially unbounded one under sustained contention — precisely what the gap path exists to prevent.
- **Let `SlotReserver` refuse a slot with `sweep_gap_count > 0`.** Localises the cost to reservation and keeps the capacity reclaimable, but needs a story for who clears the flag, and puts a new predicate on the hot candidate query. It also leaves the registry saying `free` about a slot that is not, which is the lie this ADR is removing.
- **Accept the residue and document it.** The cheapest option, and the one the blueprint already half-states. Rejected because it contradicts `0009` step 4 and its "structurally impossible" consequence, so taking it would mean amending the normative parts of an `Accepted` ADR to match an implementation accident.
- **Substitute `$newCursor` for `$cursor + $chunkSize` without `max()`.** Behaviourally equivalent in every case that terminates, per Decision 1 — it was rejected for being *less* conservative, not for closing a hang. An attempt to pin the difference with a test could not be made to fail against the plain substitution, and was deleted rather than kept as a test that cannot fail.
- **Reset the cursor to zero rather than to the first gap.** Simpler, and it is what ADR `0045` made expressible, but it discards every clean chunk before the gap on every pass.

## Related

- ADR `0009` — Tombstone-Based Slot Eviction over Immediate Reuse. **Both changes restore it rather than amend it**; a dated pointer records that its step 4 and its deadlock policy were not both being honoured. Its lifecycle, chunking, singleton model, ordering and crash-resumption property are untouched.
- ADR `0045` — Tombstoning Resets a Slot's Sweep Annotations. The immediate predecessor: it closed the recycled-cursor route to the same residue and was explicitly scoped to "a sweep that completed without taking the gap path". This closes the remainder. `0045`'s rewind-friendly semantics — a cursor of `NULL`/`0` meaning "start from the beginning" — are what make decision 4 expressible in one clause.
- ADR `0029` — The Liberator's sweep omits the tenant predicate. The precedent for the shape of this ADR: a later ADR refining `0009` on one point while leaving it in force, rather than superseding it.
- ADR `0007` — Write-availability fallback. `UnmappedFieldReserver`'s narrow backfill is why gap residue does not self-heal.
- ADR `0013` — JSON payload as system of record. The basis of the blueprint's "accept the gap" argument, and correct as far as it goes — it is about the departing field's data, not the arriving field's.
- `blueprints/liberator_daemon.md` — AC#6 and AC#8 amended, and the "accept the gap" note corrected.
