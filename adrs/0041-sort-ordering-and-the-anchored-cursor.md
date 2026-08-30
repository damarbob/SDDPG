# 0041 - Sort Ordering and the Anchored Cursor

**Status:** Proposed
**Created:** 2026-08-30

## Context

Until now every read in the engine was `ORDER BY entry_data.id ASC`, hard-coded in the probe compiler and again in the bounded fetch, and neither `EntryQuery` nor `SearchRequest` carried a sort parameter. A consumer got insertion order or nothing: no "newest first", no sortable column header. For a library whose premise is user-defined fields rendered in someone's UI, that was the largest remaining consumer-facing gap.

Whether the omission was ever *decided* is the interesting part, and the answer is that half of it was. [`queryfilter_wire_format.md`](../blueprints/queryfilter_wire_format.md) assigns "sort field and sort direction" to the caller's domain and records "sort, cursor, and pagination keys are not defined here; they belong with the consumer-facing API layer (StarGate). Confirmed 2026-05-02." That settles what goes *inside* a QueryFilter payload. It never settled whether the engine's read API accepts a sort, and it delegated to a consumer layer with nothing to delegate to.

Meanwhile four documents assume the engine does accept one. [`architecture_blueprint.md`](../architecture_blueprint.md) §1.2 — "any function-API attempt to filter **or sort** on a field lacking an explicit `is_filterable = true` registry flag immediately aborts at the API boundary with a typed exception". [`0004`](0004-fail-fast-on-unindexed-filters.md), which is **Accepted** — "will reject filters **and sorts** on fields that are not explicitly provisioned as queryable". [`glossary.md`](../glossary.md) repeats it. And [`0006`](0006-cursor-based-pagination.md) lists "a cursor is invalidated if the caller changes sort order" as a consequence, which presupposes a parameter that can be changed.

ADR `0004`'s sort clause was therefore **vacuous**: there was no sort to reject. This ADR makes it real rather than amending it.

Two things had been recorded as blocking. `0006` scopes the cursor to a single integer, which supports stable keyset pagination only when that key drives the sort. And [`0030`](0030-string-slot-storage-type-and-index-prefix.md) §51 states that `max_sort_length` truncates an `ORDER BY` on a TEXT slot at 1024 bytes, and requires revisiting before any sort-on-field feature ships. **The second turned out to be false.** See "The `0030` correction" below.

## Decision

### The parameter

`EntryQuery` and `SearchRequest` each gain an appended `?SortSpec $sort = null`. `null` means `entry_data.id ASC` — the ordering every read already had — so no existing caller changes behaviour and no existing cursor is invalidated.

`SortSpec` names one of three targets:

| Target | Orders by | Cost |
| :-- | :-- | :-- |
| `Id` | `entry_data.id` | index range scan, or a backward index scan for `DESC` |
| `CreatedAt` | `entry_data.created_at` | range scan on `(tenant_id, deleted_at, created_at)` |
| `Field` | a registered field's indexed slot column | temporary table + filesort over the filtered set |

**One sort key, not a list.** `entry_data.id` is always appended in the same direction as an implicit tiebreak, which is what makes every ordering total and therefore what makes the cursor stable. A second caller-supplied key multiplies the keyset predicate's NULL-branch surface for very little gain; it is deferred.

Sort does **not** enter the QueryFilter wire format, per the 2026-05-02 confirmation above. It is a parameter on the read DTOs. `queryfilter.schema.json` and the 13-code validation taxonomy are untouched.

### Sortability is a driver capability

`EntrySearchInterface` gains `supportsSortOn(int $fieldId): bool`, per [`0022`](0022-search-driver-capability-jurisdiction.md)'s "one method per coarse capability". `MysqlNativeDriver` answers it with the same live-indexed-slot test as `supportsFilterOn()`; the two are separate methods because they coincide only there — an external engine may index a field for matching without keeping it orderable.

Rejections: `UnknownFieldException` for a sort naming no registered field (the identical fact a filter reports, and not worth a second class), and a new `FieldNotSortableException` when the driver declines. Both emit the existing `pre_flight_rejected` event with new `reason` values — `sort_field_unknown`, `sort_field_not_sortable`, `cursor_sort_mismatch`. No new event *names*, so [`0020`](0020-structured-logging-mandate.md) needs no amendment.

Intrinsic targets never reach the driver. They are columns of the core table, not fields, and no driver may decline them.

### The anchored cursor

A cursor names the last row of the previous page **and the ordering that page was walked in**. It does not carry that row's sort *value*: the compiler resolves the value from the anchor row at query time via a one-row derived table.

```sql
CROSS JOIN (SELECT (SELECT <col> FROM <table> WHERE entry_id = ? AND tenant_id = ?) AS av) sort_anchor
```

Written as `SELECT (SELECT …)` rather than `SELECT … FROM …` deliberately: the inner form yields **zero** rows when the anchor has no row on that page, and a `CROSS JOIN` against zero rows annihilates the result set. The scalar-subquery form always yields exactly one row, NULL when there is nothing to find — which the predicate below already handles as the NULL block.

Embedding the value instead was considered and rejected on size: a string slot holds 4096 characters, so a self-contained token would reach roughly 22 KB, past every practical URL and header limit. The anchored form keeps a token constant-size and keeps `0006`'s "cursor derived from the `id` of the last record seen" **literally true**.

Wire format: unsorted reads keep emitting `base64url("v1:" . entryId)` byte-for-byte. A sorted read emits `base64url("v2:" . json)` carrying the anchor id, the sort key identity, and the direction. Both decode. A v1 token is accepted by an explicitly-default sort, because `null` and `byId(Asc)` name the same ordering.

Stamping the ordering is what finally makes `0006`'s "a cursor is invalidated if the caller changes sort order" **enforceable** — it is now a typed `InvalidCursorException` rather than a documented hazard.

### The keyset predicate

For sort key `k` and anchor value `av`, ascending:

```sql
(   (av IS NULL     AND k IS NOT NULL)          -- NULLs sort first; from a NULL anchor, every non-NULL row is ahead
 OR (av IS NOT NULL AND k > av)
 OR (k <=> av AND entry_data.id > ?) )          -- NULL-safe tiebreak, so the NULL block itself is walkable
```

Descending mirrors it: the first branch becomes `av IS NOT NULL AND k IS NULL`, because NULLs sort last.

**All three branches are load-bearing.** A slot column is nullable — a row whose value has not been mirrored into its slot (an [`0007`](0007-write-availability-under-slot-exhaustion.md) exhaustion fallback still awaiting the Reconciler) reads NULL — and `k > NULL` is UNKNOWN, so a two-branch predicate silently loses every NULL row. Verified by deleting the first branch and confirming an ascending walk stops at the end of the NULL block and never reaches the valued rows.

That a NULL slot means "not mirrored yet" rather than "no value" is **the same visibility gap filters already have** under `0007`: such a row does not match a filter on that field either. It is stated here as existing behaviour, not introduced by this ADR.

### The `0030` correction

`0030` §51 states that `max_sort_length` (default 1024 bytes) truncates an `ORDER BY` on a TEXT slot, "so an `ORDER BY i_str_NN` would compare only the first 1024 bytes", and requires this ADR to revisit it. **Probed on MySQL 8.0.13 on 2026-08-30, that is not what happens.**

```text
3 rows, identical for the first N bytes, differing only after:
  N = 1025 bytes   ORDER BY val ASC -> exact
  N = 4001 bytes   ORDER BY val ASC -> exact
  N = 36856 bytes (divergence in the FINAL byte) -> exact
  same, in the joined production shape with the 766-char prefix index -> exact
  2000 rows spilling past sort_buffer_size to an on-disk merge -> exact
  SET SESSION max_sort_length = 8 -> STILL exact
EXPLAIN confirms "Using filesort" in every case above.
COUNT(DISTINCT val) and DISTINCT also exact (0030 §44 is correct).
```

`max_sort_length` does not truncate `ORDER BY` on a TEXT column in these shapes. String slots are therefore ordered **exactly, on the full value**, with no collation key and no prefix truncation.

This is a factual correction to an `Accepted` ADR and changes what code someone would write — a truncating design was drafted and discarded on this evidence — so it lands as a new ADR with a dated pointer added to `0030`, per the append-only discipline. `0030` §44 and the prefix-index decision itself are unaffected.

### Scoping `0005`

[`0005`](0005-two-query-bounded-read-path.md) guarantees a two-query read in which each query touches a bounded number of rows. That is now **true of query 2 always, and of query 1 only for intrinsic sorts.** Measured on 8.0.13 against the real compiled SQL, 5 000 rows:

| Shape | Plan | Rows examined |
| :-- | :-- | :-- |
| `id ASC` (baseline) | `range` on `i_tenant_model` | bounded |
| `id DESC` + cursor | `range` on PRIMARY, **backward index scan** | 499 |
| `created_at ASC` + cursor | `range` on `i_tenant_deleted_created` | 500 |
| field sort + cursor | **`Using temporary; Using filesort`**, `entry_data` type `ALL` | 5 000 |

The anchor lookup itself is free: it plans as `SUBQUERY … const` inside a `system` derived table — evaluated **once**, not per row. That is why the uncorrelated form (binding `tenant_id` rather than correlating to `entry_data.tenant_id`) is required and not merely tidier.

A field sort therefore discovers each page by scanning the whole filtered set. The query count is unchanged at two, and materialisation stays bounded to `pageSize` rows. Ordering by a slot column cannot use its index while the query leads with `entry_data`, so this is inherent to the join order rather than a missing optimisation.

## Consequences

**Positive:**

- ADR `0004`'s sort clause, vacuous since it was accepted, is now enforced. The same is true of `architecture_blueprint.md` §1.2 and `glossary.md`.
- ADR `0006`'s cursor-invalidation-on-sort-change consequence becomes a typed error instead of a documented hazard.
- "Newest first" — the dominant consumer ask — costs a backward index scan and nothing else. So does `created_at`.
- Unsorted callers are untouched: same SQL shape, same cursors, byte-identical tokens.
- Cursor size is constant regardless of the sorted field's width.
- `0030`'s open caveat is closed with a measurement rather than carried forward.

**Negative:**

- A field sort is not bounded-discovery. It filesorts over the filtered set on every page, so deep pagination under a field sort costs materially more than under an intrinsic sort. Operators should expect `Using temporary; Using filesort` in the slow log for these and should not treat it as a regression.
- `EntrySearchInterface` grew a method, which breaks any third-party driver. This is deliberately taken **now**, while v0.3.0 is untagged and no such driver can exist outside this repository; after the tag it would be a breaking change.
- Sorting reads the slot column, so a row awaiting an `0007` backfill sorts as NULL. Consistent with filters, but a caller sorting a freshly-bulk-loaded model may see rows clustered in the NULL block until the Reconciler drains.
- A cursor's anchor row can be removed, or its slot nullified, mid-pagination. The anchor then reads NULL and the walk resumes from the NULL block rather than erroring. Bounded and rare, but not invisible.
- One more `ORDER BY` shape to keep the probe and the bounded fetch agreeing on. The fetch no longer sorts at all; it reorders in PHP against the probe's id sequence, which is exact for every mode and removes a redundant database sort — but it is now the single place that must not drift.

**Deferred, deliberately:**

- Multi-key sort.
- Sorting on a `backfilling` slot (rejected identically to a filter, for the same reason: the column is half-written).
- Index-driven ordering for field sorts, which would require the query to lead with the page table.
