# 0037 - Field Deletion Lifecycle

**Status:** Proposed
**Created:** 2026-08-24

## Context

StarDust's definition layer can register a field, rename it ([`0036`](0036-entry-payload-keys-are-field-names.md)), change its type ([`0016`](0016-field-type-change-lifecycle.md)) and turn indexing on and off ([`0034`](0034-non-filterable-fields-are-json-only.md)) — but it cannot remove one. Dynamic user-defined fields are the premise of the library, and a definition layer that cannot remove a field is not something a UI can ship against.

Deletion is not undesigned at the *slot* level. ADR [`0009`](0009-tombstone-based-slot-eviction.md) opens on it — *"When a field is deleted or demoted … its mapping in the schema registry is **immediately severed**"* — and both ADR [`0017`](0017-schema-registry-as-coordination-contract.md)'s slot state machine and `schema_reference` §4.5 carry `assigned → tombstoned : field deleted` as a normative transition. What has never been designed is the *operation*: its entry conditions, what happens to the field's stored values, and what happens to lifecycle checkpoints that outlive their field.

Three findings, measured against a real MySQL 8.0.13 on 2026-08-22 rather than read off the DDL, set the shape.

**Deletion is not gated on the Liberator.** `fk_slot_assignments_field` declares no `ON DELETE` clause, so it is `RESTRICT`, and deleting a field that holds an `assigned` slot fails with errno 1451. But the two-step tombstone already used by the retype/demote path clears `field_id = NULL` *before* flipping `status`, which releases the foreign key inside the same transaction — after which the identical DELETE succeeds immediately, with no sweep in between. The orphaned tombstone remains fully sweepable, because the Liberator's batch query keys on `status` + `page_id` + `slot_column` and never joins `stardust_fields` (ADR [`0029`](0029-liberator-sweep-omits-tenant-predicate.md)).

**Under ADR 0034 the common case holds no slot at all.** A non-filterable field is JSON-only, so deleting one involves no slot, no tombstone and no daemon. For a UI that is most deletions.

**The values do not go away with the registry row.** ADR 0036 fixes `entry_data.fields` keys as field names and records that a payload key with no matching registry row is *preserved verbatim* — stored, returned by the point read, and written into JSON export artifacts. A registry-only DELETE therefore leaves the deleted field's values consumer-visible indefinitely. That is precisely the "dead key in every payload" that makes the existing demote-and-leave workaround inadequate, so a deletion that stops at the registry does not actually delete anything.

## Decision

**Field deletion is severance now, purge later, registry row last.**

`deleteField(tenantId, fieldId)` commits one registry transaction and returns. It does not wait for anything.

1. `stardust_fields.deleted_at` is set and `is_filterable` is cleared, in one UPDATE.
2. Any live slot is tombstoned by the two-step sequence above, releasing the foreign key.
3. Terminal `rename_field_{id}` / `retype_field_{id}` checkpoint rows are removed.
4. `stardust_schema_version` is bumped exactly once.
5. A `running` `delete_field_{id}` checkpoint is opened.

From that commit the field is gone from every first-class surface: reads and introspection omit it, filters against it are rejected as unknown, new CSV export headers omit it, and writes still carrying its name have the value stripped from the inbound payload.

The Reconciler then drains the checkpoint as a fifth work source, removing the field's key from `entry_data.fields` in bounded chunks. **The final chunk hard-deletes the `stardust_fields` row, deletes the checkpoint, and bumps the schema version, in one transaction.**

Six rules are binding on the implementation.

**A non-null `deleted_at` means exactly "a deletion is in flight for this field".** It is the direct analogue of `previous_name`, on the same table for the same reason — the read and write paths already SELECT it with no join, so the predicate is free on both hot paths. It is a marker for the drain window, **not a soft-delete tier**: there is no undelete, nothing retains the row afterwards, and the purge's final chunk removes it.

**The registry row must outlive the purge.** The work source needs the field's name, model and tenant to build its JSON path, and `backfill_checkpoints` has no column for any of them. Keeping the row alive under `deleted_at` is what lets the claim query recover all three from a JOIN it already performs. This is the whole reason the DELETE is deferred; it is not deferred because the foreign key forces it.

**`is_filterable` is cleared in the same UPDATE, and this is load-bearing.** A field with `is_filterable = 1` and no live slot *is* demand: the Watcher's `PendingDemandReader` would provision a page for it (ADR [`0035`](0035-usable-capacity-is-a-satisfiability-test.md)), and the ADR [`0007`](0007-write-availability-over-query-completeness.md) exhaustion path would reserve it a fresh slot — re-taking the foreign key and making the final DELETE fail permanently, since nothing retries it. Both gate on `is_filterable = 1`. The registry readers additionally carry an explicit `deleted_at IS NULL`, because the consequence of missing one is silent and unbounded.

**A key belonging to a deleted field must be stripped from inbound writes, not merely left unmapped.** This is the rule most easily got backwards. Omitting the field from `LiveSlotMap` makes its key an *unknown* key — and ADR 0036 records that unknown keys are stored verbatim. A client still sending the name would therefore write it back into every new entry indefinitely, including rows the purge cursor has already passed and will never revisit, so the deletion would never converge. `LiveSlotMap::canonicalise()` strips it; that strip is what makes a single forward pass sufficient.

**A field may have at most one lifecycle in flight, and the order of the guards is fixed.** ADR 0036 established this for rename and retype; deletion makes it three, checked **rename → retype → delete** in every initiator so two concurrent initiators cannot each see the other's row as absent. A delete is *refused* while a rename or retype is running rather than cancelling it: both sibling repositories recover the field id by INNER JOIN on a substring of `job_name`, so deleting the field would strand a `running` checkpoint that no worker can claim and no dashboard can explain. The rename/retype-side guards key on `deleted_at` rather than on a checkpoint row, so they still fire for a field whose purge checkpoint was manually failed. They sit in the initiators rather than on the facade, following ADR 0036's placement ruling. Note compaction is **not** the back door here that it is for rename: a deleting field's slot is tombstoned and its `is_filterable` cleared, and `CompactionRepository` selects on both, so the planner cannot see it at all. The guard's value against compaction is limited to the plan-then-delete race.

**The name is not reusable until the purge lands.** `ux_fields_model_name` is unconditional, so a field being deleted still holds its name. `SchemaBuilder`'s get-or-create lookup excludes soft-deleted rows and the subsequent insert raises a typed `FieldDeletionInProgressException`. The alternative — silently returning the id of a field whose values are being erased and whose row is about to be dropped — is a data-loss bug wearing the costume of a convenience.

### What is deliberately not bridged

`get()` strips the deleted key from the payload, so it agrees with `read()` for the whole window. **The JSON export artifact does not**, matching the identical ADR 0036 carve-out: it streams the payload as stored, and a JSON consumer sees the residue. A raw `entry_data` dump likewise shows values the API has already stopped serving.

### Relationship to prior ADRs

No `Accepted` ADR is amended. ADR 0009 and ADR 0017 already name field deletion as a slot-eviction trigger and a legal state transition; this ADR supplies the operation they assumed. ADR 0020 is amended in place — it is `Proposed` — to add `delete_started` and `delete_complete` to the `registry` source vocabulary. ADR 0012 is untouched: deletion never alters page DDL, and a deleted field's column becomes free inventory after the Liberator sweep.

## Consequences

**Deletion is honest.** After `delete_complete` the field has no registry row, no slot, no checkpoint, and no key in any payload. That is a stronger guarantee than the demote-and-leave workaround it replaces, and it is what makes the definition layer usable behind a UI.

**Deletion needs a running Reconciler to finish.** Without one the field stays permanently severed-but-unpurged: invisible through the API, still present in storage, still holding its name. That is a safe resting state rather than a corrupt one — but it is a new operational dependency on an entry point that a consumer may reasonably expect to be synchronous, and the API documentation has to say so plainly.

**The name-reuse window is a real cost.** Delete-then-recreate-with-the-same-name, a plausible UI flow, fails with a typed exception until the purge completes. Freeing the name at initiation was considered and rejected: it would require renaming the field to a sentinel and stashing the real name in `previous_name`, which means "a rename is in flight" to five bridging surfaces and would make them alias the sentinel back.

**This is the first place in the engine that deletes from `backfill_checkpoints`.** ADR 0036's `insertOrReset()` rationale rests on the observation that nothing ever did. The rationale survives — the delete repository uses the same upsert, for the failure path — but the invariant does not, and the note in `src/Rename/CLAUDE.md` is corrected accordingly.

**A fifth Reconciler work source is appended, not inserted.** Round-robin order is observable in event streams. Ordering between the backfill sources is immaterial because the cross-guards make them mutually exclusive per field.

**`deleteModel()` gets easier.** It inherits the checkpoint guard, the shared tombstoner, and the severance pattern. It still has to settle its own open question — what happens to `entry_data` rows left carrying a dangling `model_id` — which this ADR does not address.

**Returning `false` rather than throwing hides the foreign-tenant case.** `deleteField()` reports "nothing to do" for a field that does not exist, belongs to another tenant, or is already being deleted. That is deliberate, matching `deleteEntry()` and the tenant-isolation rule that a caller must not be able to probe another tenant's ids — but it does mean a typo in a field id is silent.
