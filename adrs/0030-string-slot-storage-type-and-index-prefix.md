# 0030 - String Slot Storage Type and Index Prefix

**Status:** Accepted
**Created:** 2026-06-12

## Context

The QueryFilter wire format pins the maximum string value length at **4096 characters** (`queryfilter_wire_format.md` §4.5 #17, §4.6 #25), enforced at decode time (`JsonFilterDecoder`) and at pre-flight (`ValueTypeValidator`, `FilterLimits::DEFAULT_MAX_STRING_LENGTH = 4096`). The schema reference, however, specified string slots only as `VARCHAR` with no pinned length, and the shipped `PageProvisioner` hard-coded **`VARCHAR(255)`**.

The two bounds were chosen independently and disagree, with a real runtime consequence (gap analysis Gap 4): the write path performs no length check, so under `STRICT_TRANS_TABLES` a 256–4096-character value written to a _filterable_ string field raises MySQL error 1406 ("Data too long") and rolls back the entire entry write — even though the value is in-bounds per the wire format and `entry_data` (the system of record, ADR [`0013`](0013-json-payload-as-system-of-record.md)) would hold it fine.

The naive fix — widen to `VARCHAR(4096)` — is **physically impossible** in MySQL. Two independent limits forbid it (both verified empirically against MySQL 8.0):

1. **Row-definition limit (errno 1118).** MySQL caps a table's row definition at **65,535 bytes**, counting every `VARCHAR` at its full declared width. A page carries 25 string slots: 25 × 4096 chars × 4 bytes (utf8mb4) ≈ 410 KB ≫ 65,535. `CREATE TABLE` fails outright. The largest _uniform_ `VARCHAR` width that fits 25 slots alongside the other 35 columns is roughly 650 characters — far short of the 4096 bound. `TEXT` escapes this limit: it is stored outside the row (under `ROW_FORMAT=DYNAMIC`) and contributes only a ~12-byte pointer to the row definition.

2. **Index key-size limit (errno 1071).** InnoDB caps an index key at **3072 bytes** under `ROW_FORMAT=DYNAMIC` (only **767 bytes** under the legacy COMPACT/REDUNDANT formats). The filterable composite `(tenant_id, slot_column)` index therefore cannot cover more than **766 utf8mb4 characters** of the slot: 766 × 4 bytes + 8-byte `BIGINT` tenant_id = 3072 bytes exactly. Any storage width above 766 characters forces a **prefix index**, regardless of column type.

`TEXT` (65,535-byte capacity) comfortably holds every wire-format-valid value: 4096 chars × 4 bytes/char worst case = 16,384 bytes. MySQL requires an explicit prefix length to index a `TEXT` column — which the 3072-byte key limit already mandates at this width anyway.

## Decision

String slots (`i_str_01`…`i_str_25`) on extension pages are **`TEXT`**, and a filterable string slot's composite index is **`(tenant_id, slot_column(766))`** — a 766-character prefix. The page DDL pins **`ROW_FORMAT=DYNAMIC`** explicitly, because the 766-char prefix depends on the DYNAMIC 3072-byte key limit and must not silently break on servers whose `innodb_default_row_format` is COMPACT.

```sql
i_str_01 TEXT NULL DEFAULT NULL,
...
KEY ix_entry_slots_page_N_i_str_01 (tenant_id, i_str_01(766)),
...
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci ROW_FORMAT=DYNAMIC
```

**Correctness of the prefix index.** For every filter operator, MySQL ranges on the 766-char prefix and then **rechecks the full column value** on the fetched row, so `eq`/`neq`/`in`/`nin`/`prefix`/`between`/range results remain exact even for values that collide in their first 766 characters (verified empirically: two 4096-char values sharing a 766-char prefix are correctly disambiguated by an `eq` filter). The fixed-width families (`int`/`num`/`dt`) are unchanged and remain fully indexed.

**Forward-only.** The change applies to newly provisioned pages. Existing `VARCHAR(255)` pages are not altered (ADR [`0012`](0012-immutable-extension-page-ddl.md) forbids DDL on populated pages); pre-`0.3.0` development databases pick up the new shape on re-bootstrap. The int/num/dt slot definitions, the 25/15/10/10 layout, and the Index Provisioning Policy (ADR [`0003`](0003-schema-driven-index-provisioning.md)) are untouched — only the string family's column type and its index expression change.

`FilterLimits::maxStringLength` stays at 4096: the bound is now honored by storage, not weakened to meet it.

## Consequences

**Positive:**

- A filterable string field accepts and correctly filters values up to the full 4096-character wire-format bound; the validator, the write path, and storage agree for the first time. The 1406 write-failure class for in-spec values is gone.
- Every wire-format-valid value fits by construction (16,384-byte worst case vs. 65,535-byte TEXT capacity) — no encoding-weight edge cases.
- `COUNT(DISTINCT slot)` (the ADR [`0019`](0019-index-cardinality-policy.md) cardinality aggregate) deduplicates on the **full** TEXT value, not a truncated prefix (verified empirically on MySQL 8.0; values identical beyond `max_sort_length` count as distinct). The cardinality advisory is unaffected.
- Short values are stored in-row under DYNAMIC (off-page storage engages only when a row actually outgrows the page), so the common short-string case pays no meaningful TEXT penalty.

**Negative / constraints:**

- **String-slot index scans are prefix scans.** A prefix index is never covering: every index hit costs a clustered-row lookup for the full-value recheck. Range scans on string slots are "prefix range + recheck" rather than the pure index scans the int/num/dt families keep. This is the necessary price of any storage width above 766 chars — `VARCHAR(4096)` (were it possible) would pay the same.
- **`ROW_FORMAT=DYNAMIC` is load-bearing.** The DDL pins it explicitly; a future edit removing the clause would fail `CREATE TABLE` with errno 1071 on COMPACT-default servers. (The previous `VARCHAR(255)` full-column index — 1020 bytes — silently carried the same latent dependency; it is now explicit.)
- **No `ORDER BY`/`GROUP BY` on string slots.** MySQL applies `max_sort_length` (default 1024 bytes) when _sorting_ TEXT, so an `ORDER BY i_str_NN` would compare only the first 1024 bytes. No current code path sorts or groups on a slot column (pagination is keyed on `entry_data.id`; the cardinality aggregate groups on `tenant_id`); any future sort-on-field feature MUST revisit this ADR.

> **Corrected 2026-08-30 by ADR [`0041`](0041-sort-ordering-and-the-anchored-cursor.md):** the bullet above is **empirically wrong on MySQL 8.0.13**, and the sort-on-field feature it defers to has now shipped. `max_sort_length` does *not* truncate an `ORDER BY` on a TEXT column in the shapes this engine emits. Probed 2026-08-30: values identical for the first 1 025, 4 001 and 36 856 bytes and differing only afterwards all ordered **exactly** — in the joined production shape with the 766-char prefix index present, across an on-disk merge sort, and with `max_sort_length` forced down to **8**. `EXPLAIN` confirmed `Using filesort` throughout, so the sort was real and simply not truncating. String slots are therefore ordered on the **full value**, and 0041 carries no collation key or prefix truncation — a truncating design was drafted against this bullet and discarded on the evidence. The `COUNT(DISTINCT)` note above is correct and unaffected, as is the prefix-index decision itself.
- **The 1406 boundary moves, it does not vanish.** A value exceeding 65,535 _bytes_ (only possible far outside the 4096-char wire bound, e.g. via the write path, which deliberately does not length-check) would still fail the slot UPSERT. The failure threshold moves from 255 chars to 64 KiB; in-spec values can no longer trip it.

**References:** ADR [`0003`](0003-schema-driven-index-provisioning.md) (index provisioning policy), ADR [`0012`](0012-immutable-extension-page-ddl.md) (immutable page DDL — forward-only application), ADR [`0013`](0013-json-payload-as-system-of-record.md) (JSON payload remains the system of record), ADR [`0019`](0019-index-cardinality-policy.md) (cardinality aggregate verified TEXT-safe), [`queryfilter_wire_format.md`](../blueprints/queryfilter_wire_format.md) §4.5/§4.6 (the 4096-char bound).

## Addendum (2026-06-16): write-path length guard closes the residual string-family 1406

The "Consequences → Negative" note above states the write path "deliberately does not length-check," leaving a residual raw-1406 risk for any string value exceeding the 65,535-byte `TEXT` capacity. That residual is now closed for the string family.

`PayloadSplitter::coerceString()` enforces the 4096-character bound on every value bound for a string slot — `mb_strlen` against `FilterLimits::DEFAULT_MAX_STRING_LENGTH`, fail-fast **before any SQL**, raising the typed `UncoercibleSlotValueException` (the same exception the BIGINT-overflow guard already uses). This matches how `JsonFilterDecoder` and `ValueTypeValidator` measure the identical bound, so the write contract is symmetric with the filter contract the slot is queried by.

**The bound is pinned to the `FilterLimits::DEFAULT_MAX_STRING_LENGTH` _constant_, not the injectable `Config::$queryFilterLimits->maxStringLength`.** This is deliberate: 4096 characters is at most 16,384 bytes (utf8mb4 worst case), a quarter of the 65,535-byte `TEXT` ceiling, so pinning to the constant makes a raw 1406 **impossible by construction**. Honoring the injected limit instead would let an operator raise it above ~16,383 characters and re-open the exact 1406 hole this ADR set out to close. The injected limit governs the QueryFilter _wire format_; the slot guard governs _storage_ and tracks the storage design constant. The one residual asymmetry is benign and in the opposite direction: an operator who _lowers_ the filter limit (e.g. to 1024) gets filters that reject >1024-char operands while writes still accept up to 4096 — such a value lands in the slot but is simply not `eq`-matchable, never a DB failure (`entry_data` remains authoritative per ADR [`0013`](0013-json-payload-as-system-of-record.md), which is intentionally not length-checked). The int/num/dt families remain guarded by their existing type-coercion overflow checks.

This addendum records an implementation refinement that _honors_ the original decision; the Decision and Context above are unchanged.
