# 0036 - Entry Payload Keys Are Field Names

**Status:** Proposed
**Created:** 2026-08-22

## Context

ADR [`0013`](0013-json-payload-as-system-of-record.md) establishes `entry_data.fields` as the sole authoritative system of record for entity field data, but it never states what the JSON object's *keys* are. In practice they are `stardust_fields.name`: the write path hands the consumer's array through to `json_encode` verbatim, and every reader looks values up by name. The convention was never decided — it emerged — and the only place it is pinned in the design corpus is an incidental JSON-path literal, `JSON_EXTRACT(fields, '$.fieldName')`, repeated in ADR 0013 §Decision, ADR [`0009`](0009-tombstone-based-slot-eviction.md), ADR [`0017`](0017-schema-registry-as-coordination-contract.md), and the architecture blueprint.

That undecided convention has a consequence: renaming a field is a rewrite of every entry's payload, not a registry write. The engine currently has no rename or delete for models or fields at all, which is the gap that makes its definition layer unusable behind a UI, and rename is the operation the key format governs.

The question, raised before tagging `0.3.0-alpha.1`, was whether to re-key the payload by `stardust_fields.id`. That would collapse a rename to a single `UPDATE stardust_fields SET name = ?`, permanently. Because the payload format is a consumer-visible contract — it surfaces through the point-read API and the JSON export artifact — the moment to change it, if ever, is before the first tag rather than after.

## Decision

**The `entry_data.fields` key is the field name, `stardust_fields.name`. This is now a decision rather than an accident, and it does not change for v0.3.0.**

Four findings drove it.

**Unknown keys have no id.** ADR 0013's completeness premise — the payload is authoritative for *all* entity field data and "must always be kept complete" — is implemented as: a payload key with no matching row in `stardust_fields` is stored verbatim, gets no slot, and is never enqueued. Such a key has no `field_id` by construction. An id-keyed payload would have to drop those values (contradicting 0013's premise), or admit a mixed key space in which a field legitimately named `17` is indistinguishable from field id 17, or introduce a nested envelope that changes the shape for every consumer. None of the three is better than the problem being solved.

**Names are the contract at three external surfaces.** The CSV export header is derived from `stardust_fields.name` and is used directly as the payload projection key; the JSON export artifact emits the payload verbatim; and the async import artifact is a file on disk whose `fields` object is name-keyed. The latter two are formats already written by prior deploys, so a key change is not confined to the codebase.

**The blast radius is concentrated in silent-failure sites.** Most payload lookups are of the form `$payload[$name] ?? null`. A key-space mismatch therefore produces empty values rather than an exception — a fully-populated CSV header over blank cells, or a backfill that NULLs every slot without emitting an event. That is the worst available failure mode for a migration.

**Filters are unaffected by the choice.** No filter compiles to a JSON predicate; every leaf resolves against an indexed slot column, keyed by `field_id` already. The key format governs reads, writes, exports and imports, but not the query path — so the performance argument that would favour ids does not arise.

Renaming a field is therefore a **chunked backfill** over `entry_data`, structurally analogous to the ADR [`0016`](0016-field-type-change-lifecycle.md) retype pipeline and riding the same `backfill_checkpoints` table and Reconciler work-source mechanism. The registry name flips immediately at initiation; the payload is rewritten asynchronously; a nullable `stardust_fields.previous_name` column carries the old name for the duration and is cleared in the same transaction that completes the backfill.

Two rules follow from that window and are binding on the implementation:

- **Reads and writes bridge the window; filters do not.** The read path falls back from the new key to `previous_name`, and the write path canonicalises an old-name key to the new name before persisting. A *filter* naming the old field is rejected outright. The asymmetry is deliberate: a rejected write loses data, a rejected filter loses nothing, and loud rejection is the correct failure mode for a query.
- **A field may have at most one lifecycle in flight.** A retype and a rename must not overlap: the retype backfill locates values by name, so during a rename window it would read every un-migrated row as "value absent" and write NULL into the slot without emitting a coercion event.

## Consequences

**Positive:**

- The payload stays self-describing. A raw `entry_data` row, a JSON export artifact, and an import artifact are all readable without a registry lookup, which is what makes the payload usable as a recovery and debugging surface.
- ADR 0013's completeness premise is preserved exactly as written; unknown keys keep working with no new policy.
- The CSV header, the JSON artifact, and the import artifact formats are unchanged, so no on-disk artifact written by an earlier deploy becomes unreadable.
- Rename reuses the retype pipeline's shape — checkpoint table, work source, chunked drain, cursor advance, final-chunk atomic completion — rather than introducing a new coordination mechanism.

**Negative:**

- A field rename costs one pass over every entry in the model, forever. It is an operator-initiated, infrequent operation, but it is O(entries) and will never be O(1).
- A rename opens a window during which the payload is heterogeneous — some rows on the old key, some on the new — which the read path, the write path, the CSV export path and the point read must each bridge explicitly. Each of those is a place a future contributor can reintroduce a silent null.
- `stardust_fields` carries a `previous_name` column whose only purpose is that window, and it introduces a name-collision case the existing `UNIQUE (model_id, name)` index does not cover: renaming a second field into a first field's vacated name would make the first field's fallback read the second field's values. Initiation must guard against `name` **and** `previous_name`.
- The point read gains a schema-version probe it did not previously need, so that it cannot disagree with the paginated read during a rename window.

**Rejected alternatives:**

- **Key the payload by `stardust_fields.id`.** Collapses rename to a single registry UPDATE. Rejected because unknown keys have no id, because two on-disk artifact formats and the CSV export contract are name-based, and because the migration's failure modes are silent rather than loud. The rename cost it removes is paid rarely; the costs it adds are paid on every read, export and import.
- **A nested envelope — `{"f": {"17": …}, "u": {"unknown": …}}`.** Preserves both the O(1) rename and the unknown-key guarantee. Rejected as the deepest change of the three: it breaks every consumer expectation and both export artifact shapes to optimise an infrequent operator action.
- **Reject unregistered keys at `write()`.** Would make an id-keyed payload viable by eliminating the unknown-key case. Rejected here because it is a separate decision about write semantics that should be argued on its own merits, not adopted as a side effect of a storage-format change; it would also turn a currently-lossless write path into a rejecting one.
- **Keep name keys and offer no rename.** The status quo. Rejected because the definition layer is the library's premise, and a schema that can only be appended to is not one a UI can present.
