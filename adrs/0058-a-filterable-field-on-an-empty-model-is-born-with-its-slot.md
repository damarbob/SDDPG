# 0058 - A Filterable Field on an Empty Model Is Born With Its Slot

**Status:** Proposed
**Created:** 2026-11-12

## Context

Registering a field as filterable records an intent and reserves nothing. `SchemaBuilder` inserts the `stardust_fields` row and stops. The slot comes after a write has touched the field, when the Reconciler claims one from the sync queue. ADR [`0007`](0007-write-availability-over-query-completeness.md) says so in a clarification dated 2026-10-29, "No reservation happens before a write". For a field registered on a model with no entries, the registry reads like this until that first write has been drained:

| table | row |
| :-- | :-- |
| `stardust_models` | `id = 7`, `name = 'article'`, no rows in `entry_data` for `model_id = 7` |
| `stardust_fields` | `id = 41`, `model_id = 7`, `name = 'title'`, `declared_type = 'string'`, `is_filterable = 1` |
| `stardust_slot_assignments` | no row with `field_id = 41` |

`describeModel()` reports `title` with `isIndexed = false`, and a filter on it is rejected, although nothing could be missing from the index of a model with no entries. The wait has no upper bound: with no write and no Reconciler tick the field stays unindexed. A consumer that lets users add fields interactively has to register the field as non-filterable, promote it in a second step, and guard every read of it.

ADR `0007` gave two reasons for not reserving at registration, and neither holds here.

- A slot would be held for a field that may never be written. The Watcher already counts every filterable field without a slot as pending demand (`PendingDemandReader`) and provisions indexed capacity for it, so reserving only names which slot belongs to the field.
- A daemon step would reverse the Reconciler's rule that it reserves only after a write. This ADR adds no daemon step, because the reservation happens inside the registration call.

One fact limits how far this can go. An unknown payload key is kept in `entry_data.fields` (ADR [`0013`](0013-json-payload-as-system-of-record.md)), so on a populated model the new field's key may already sit in payloads, and a fresh slot would be empty for them. Only a model with no entries is guaranteed to have nothing to backfill.

## Decision

When `SchemaBuilder::createModel()` or `SchemaBuilder::defineField()` inserts a new filterable field on a model that has no entries, it reserves an indexed slot for the field in the same transaction, so the field is queryable when the registration commits.

**What counts as having no entries.** No row in `entry_data` for the field's `tenant_id` and `model_id`, whatever its `deleted_at`. The check is `SELECT 1 FROM entry_data WHERE tenant_id = ? AND model_id = ? LIMIT 1` inside the registration transaction, a lookup on `ix_entry_data_tenant_model` that stops at the first row. A model created by the same `createModel()` call has no entries by construction and skips it. Soft-deleted rows count, because they still carry their payload and a later restore should not force this rule to be revisited.

**What is reserved.** One free slot of the field's type family on a page whose column is indexed, using the same `IndexedSlotPredicate` as the Reconciler's reservation so ADR [`0004`](0004-fail-fast-on-unindexed-filters.md) holds. It is taken as `assigned`, the status the exhaustion path gives a reserved slot and one that is queryable at once. ADR [`0032`](0032-model-affine-slot-reservation.md) affinity applies unchanged, and because the reservation shares the registration transaction it sees the slots that earlier fields of the same `createModel()` call took, so one new model's fields land together on a page. After the change the registry reads:

| table | row |
| :-- | :-- |
| `stardust_fields` | `id = 41`, `model_id = 7`, `name = 'title'`, `declared_type = 'string'`, `is_filterable = 1` |
| `stardust_slot_assignments` | `page_id = 3`, `slot_column = 'i_str_04'`, `slot_type = 'str'`, `field_id = 41`, `status = 'assigned'` |

`describeModel()` now reports `isIndexed = true`, a filter on `title` is accepted and returns an empty result, and the first write puts its value in `i_str_04` inside the write transaction with no queue row and no Reconciler.

**It is best effort and never fails a registration.** With no indexed free slot of the family, the field is registered without one, exactly as today. The Watcher already sees it as pending demand, and the existing path takes over. No DDL runs at registration, so ADR [`0016`](0016-field-type-change-lifecycle.md) commitment 4 is intact, and registration never waits for capacity, in the spirit of ADR `0007`.

**A populated model changes nothing, and neither does an existing field.** If the probe finds any entry, the field is registered as it is today and gets its slot after a write. A consumer who needs existing values indexed keeps using `promoteFieldToFilterable()`, which backfills. `defineField()` stays get-or-create, so a field that already exists is returned as it is and no installed field is migrated.

**One schema version bump.** `createModel()` keeps its documented rule of bumping `stardust_schema_version` once when it inserts anything, however many slots it reserved. The reservation therefore needs a variant that leaves the bump to its caller, because `SlotReserver` otherwise bumps on its own.

**Events.** After the commit the registration emits `slot_reserved` for each reserved slot, with `status = 'assigned'` and a correlation id minted for the registration. The event name exists already, so ADR [`0020`](0020-structured-logging-mandate.md) is unchanged.

**Surface and setting.** `Config` gains one trailing boolean, `reserveSlotAtRegistration`, which defaults to `true`. With `false`, registration behaves exactly as it did before this ADR. `StarDust` gains nothing. `SchemaBuilder` takes the reserver as an optional trailing constructor argument, a `SchemaBuilder` built without one behaves as it does today, and `StarDust::schemaBuilder()` passes it only when the setting is on.

### The race this does not close

The probe reads a consistent snapshot. A write that commits after that snapshot but before the registration commits, with the field's key already in its payload, would leave that entry with the value in its JSON and nothing in the slot. The same hole exists today for any field registered after a client began sending its key, and the lazy path never backfilled those entries either. A write that does not send the key has nothing to index, so the usual case is not affected. This ADR neither widens the hole nor closes it.

## Consequences

**Positive:**

- A filterable field on a new model is indexed from the moment it exists. A consumer needs no promotion step and no guard on `isIndexed` for that case, and a filter control can be built from the field list directly.
- The first writes to a new model fill their slots in the write transaction. Before, each of those writes queued a sync row and waited for a Reconciler to reserve and backfill, and a host with no Reconciler running never got the index.
- Registration order becomes page layout. The fields of one new model are reserved in one transaction, so affinity places them together, which ADR `0032` aims for and reaches only by luck when reservations come one write at a time.

**Negative:**

- A slot is taken at definition for a field that may never be written. The cost is bounded by the field count and is capacity the Watcher was already provisioning for the same field as demand.
- Registration locks one free slot row per filterable field (ADR `0032` measured one row per reservation) and holds the schema version row for the length of the transaction, as it did already. A lock timeout or deadlock in the reservation rolls back the whole registration, and the caller retries it like any other registry statement.
- Behaviour a consumer can see changes. A filter on a just-registered field of an empty model was rejected and is now accepted. The README and `docs/configuration.md` say the opposite today and change with the code, along with the smoke tests that pin the lazy behaviour. Setting `reserveSlotAtRegistration` to `false` restores the old behaviour.
- A filterable field added to a populated model still has no slot until its first write, so consumers keep gating on `isIndexed` for that case.
- The race above stays.

**Rejected alternatives:**

- Keep reserving after a write and tell consumers to reserve by hand. The manual route needs `PageProvisioner` and `SlotReserver`, both internal classes, so it is not available to a consumer of the public surface.
- Reserve on every model, opening a backfill checkpoint on a populated one. Correct for all models, but registration would start a Reconciler drain over every entry and `SchemaBuilder` would orchestrate the retype pipeline, which is the promotion path under another name. The field would also stay unindexed during the drain.
- Scan payloads for the key instead of checking for entries. Exact, and it would let a populated model qualify, but it reads every payload of the model inside the registration transaction, so the cost grows with the model.
- Reserve from a Watcher step when it provisions a page. It cannot be immediate, since the Watcher runs on its own cycle, and it is the daemon step ADR `0007` argued against.
- Take the slot as `ready`. That status means a backfill finished, while `assigned` is what a reservation with nothing to backfill produces everywhere else, and both are queryable.
- Make `promoteFieldToFilterable()` finish in one transaction on an empty model too. The same proof would allow it, but a promotion is operator-initiated, already reserves at once when capacity exists, and completes on the next Reconciler tick, so it is left out until someone needs it.

## Related

- ADR `0007` - Write Availability Over Query Completeness (its 2026-10-29 clarification is the decision this ADR narrows)
- ADR `0004` - Fail-Fast on Unindexed Filters (the reserved slot must be an indexed one)
- ADR `0013` - JSON Payload as System of Record (why an unknown key can sit in a payload, and so why only an empty model qualifies)
- ADR `0016` - Field Type Change Lifecycle (commitment 4, no eager DDL, is unchanged, and the promotion path remains the way to index a populated model)
- ADR `0020` - Structured Logging Mandate (no event name is added)
- ADR `0022` - Search Driver Capability Jurisdiction (the `field_not_filterable` rejection of a field with no usable slot still applies to a populated model)
- ADR `0032` - Model-Affine Slot Reservation (affinity applies to the eager reservation unchanged)
- ADR `0034` - Non-Filterable Fields Are JSON-Only (only a filterable field is reserved a slot)
