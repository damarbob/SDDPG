# 0057 - String Slots Are Indexed Through a Generated Key

**Status:** Proposed
**Created:** 2026-10-28

## Context

A filterable string field holds a string slot, which is one `TEXT` column on an extension page such as `i_str_01` on `entry_slots_page_1`. [`0030`](0030-string-slot-storage-type-and-index-prefix.md) indexes it as `(tenant_id, i_str_01(766))`. InnoDB caps an index key at 3072 bytes, so this is a prefix index that covers only the first 766 characters of each value. It can narrow a search to candidate rows, it cannot prove that a row matches, and the server rechecks the full value on every row it finds. Three problems with that shape have been measured since 0030 was accepted.

A text equality cannot be answered in `entry_id` order. [`0056`](0056-filtered-default-order-reads-drive-from-a-full-width-equality-leaf.md) lets a filtered read start from an equality on an integer, numeric or datetime slot and stop after one page. It leaves text equality out, because a prefix index does not return equal values in `entry_id` order, so on MySQL the read touches every matching entry and sorts them to find one page. Measured with session handler counters (the server's own count of rows read and rows sorted for one statement) on 200,000 rows in two tenants with a page size of 50, an equality matching 5,004 entries of one tenant touched 10,009 rows on MySQL 8.0.46 and 15,065 on MySQL 8.0.13, and sorted 51 on both. MariaDB 10.11 and 11.8 already stop early on that data, at 1,727 rows touched and none sorted. Started from a full-width key on the page, the same read touched 102 rows and sorted none on all four engines (runs of 2026-10-18).

A row that holds several long values can be inserted and then not updated or deleted. InnoDB keeps an undo record for every change, a copy of the row's earlier state so that the statement can be rolled back, and for an indexed string stored off the row's page that copy includes up to 766 characters of the value. When the record outgrows its limit the statement fails with errno 1713, "Undo log record is too big". On 16 KB pages, which is the default, an update of an integer field in a row with 25 string slots failed once more than 16 of the slots held 800 ASCII characters, once more than 4 held 4,096 ASCII characters, and once more than 5 held four-byte characters, on MySQL 8.0.13 and 8.0.46. Inserting those rows always worked. In the write runs with mixed value lengths (mostly short, 10% of 200 characters and 2% of 2,000), today's page failed with 1713 in all 7 paired repetitions for the update on MySQL and for the null-out and the cascade delete on MariaDB 10.11 and 11.8 (runs of 2026-10-05 and 2026-10-19).

A value can be missed. A search through the prefix index uses the first 766 characters of the search value, and a collation (the rule set that decides which strings are equal) can call two values equal although their first 766 characters differ. Under `utf8mb4_0900_ai_ci`, `ß` equals `ss`. On MySQL 8.0.13, a value of 766 `ß` followed by `x` and a value of 1,532 `s` followed by `x` are equal, yet a search for either one through the index returned only that row. The same search with the index ignored returned both (checked 2026-10-28).

One more constraint shapes the fix. A populated page is never altered ([`0012`](0012-immutable-extension-page-ddl.md)), so whatever replaces the prefix index applies to pages built after it, and the engine has to serve several page shapes at once.

## Decision

A page built from this release on carries, for every string slot, a stored generated column that holds a key derived from the value's weight string, and indexes that key in place of the 766-character prefix. A lookup on a string slot is answered from the key and then rechecked against the value. The engine reads a page's shape from the page's own table definition, so prefix pages and key pages are served side by side.

### The key

Three terms first. A collation is the rule set that decides which strings are equal and how they sort. The page collation is `utf8mb4_0900_ai_ci` on MySQL and `utf8mb4_unicode_520_nopad_ci` on MariaDB, and both call `paid`, `Paid` and `païd` equal. A weight string is the byte string a collation really compares, so two values the collation calls equal have the same weight string, and comparing weight strings byte by byte reproduces the collation's order. A generated column is a column the server computes from other columns on every write, and `STORED` means the result is saved in the row.

Each string slot `i_str_NN` gets a companion `i_str_NN_k VARBINARY(48)`. It holds the first 32 bytes of the slot's weight string, padded with zero bytes up to 32 (the head), followed by the first 16 bytes of the SHA-256 digest of the whole weight string (the digest). These are real values from MySQL 8.0.13 under `utf8mb4_0900_ai_ci`.

| `i_str_01` | weight string | `i_str_01_k` head (32 bytes) | `i_str_01_k` digest (first 8 of 16 bytes) |
| :-- | :-- | :-- | :-- |
| `paid`, `Paid`, `païd` | `1E0C1C471D321C8F` | the weight string, then 24 zero bytes | `1C784DE371F5B22D` |
| `pad` | `1E0C1C471C8F` | the weight string, then 26 zero bytes | `6F7EE8857FF9CA78` |
| `ß`, `ss` | `1E711E71` | the weight string, then 28 zero bytes | `905373FCD62FAA2D` |

The key is NULL exactly when the slot is NULL, and the empty string gets a key of its own. For `i_str_01 = 'PAID'` the compiler turns the value into the same 48 bytes, finds them in the index `(tenant_id, i_str_01_k)`, which returns equal keys in `entry_id` order, and then rechecks `i_str_01 = 'PAID'` on the row.

Each part of the key has one job. The head keeps byte order for the first 32 bytes, so a prefix or a range bound can still narrow the index. The digest separates values that share their head, which makes an equality exact at any length, and the chance that two different weight strings share the same 16 digest bytes is negligible, with the recheck dropping such a row anyway. The zero padding keeps a short value in the right place against longer ones. Without it the `lt` and `lte` bounds returned the wrong rows for 158 of 165 probes and `between` for 53 of 165 on MySQL 8.0.13 and 8.0.46 (probe run of 2026-10-18).

### The page

```sql
i_str_01   TEXT NULL DEFAULT NULL,
i_str_01_k VARBINARY(48) GENERATED ALWAYS AS (
    CONCAT(
        RPAD(LEFT(WEIGHT_STRING(i_str_01), 32), 32, UNHEX('00')),
        LEFT(UNHEX(SHA2(WEIGHT_STRING(i_str_01), 256)), 16)
    )
) STORED,
...
KEY ix_entry_slots_page_N_i_str_01 (tenant_id, i_str_01_k),
```

The column is declared after the slot columns, the index keeps its name, and `ROW_FORMAT=DYNAMIC` stays. A page still has one index per string slot, so a full page still carries 62 keys against the server's ceiling of 64. The write path never names the key column, so the server fills it on every insert and update, including the padding in a multi-row upsert. The Liberator's `UPDATE ... SET slot = NULL` and the backfills recompute it the same way.

One class owns the layout. `StringKeyLayout` holds the head width, the digest width, the column expression and every lookup predicate below, because a lookup that repeats the layout even slightly wrong returns wrong rows without raising an error. A test fails when `WEIGHT_STRING(` or `SHA2(` appears anywhere else in `src/`.

A server whose `SHA2` returns NULL would give every row a NULL key and every equality zero rows, with no error. For a page with a string slot, `provision()` therefore checks before any DDL that `SELECT SHA2('x', 256)` returns 64 characters, and refuses with `UnsupportedServerException` otherwise. A page with no string slot has no key to compute and does not ask.

### How a lookup uses the key

This applies to the join strategy only, the one used when a filter has no `or` and no `not`. The EXISTS strategy reaches the page by primary key for each entry, so a key adds nothing there and string leaves keep today's SQL. A rewritten leaf is `{key predicate} AND {original predicate}`. The original always stays as the recheck, so the key only has to find a superset of the true matches. In the table `w` is `WEIGHT_STRING(CONVERT(? USING utf8mb4) COLLATE <page collation>)`, `H` is the head width read from the page (32) and `k` is the key column.

| Filter | Key predicate added |
| :-- | :-- |
| `eq` | `k = CONCAT(RPAD(LEFT(w,H),H,UNHEX('00')), LEFT(UNHEX(SHA2(w,256)),16))`, with the value bound twice |
| `in` | the same expression for each element, as `k IN (...)` |
| `prefix` | `k BETWEEN LEFT(w,H) AND CONCAT(LEFT(w,H), REPEAT(UNHEX('FF'),H+16))`, built from the unpadded head |
| `gt`, `gte` | `k >= RPAD(LEFT(w,H),H,UNHEX('00'))` |
| `lt`, `lte` | `k <= CONCAT(RPAD(LEFT(w,H),H,UNHEX('00')), REPEAT(UNHEX('FF'),16))` |
| `between` | both bounds above |

`neq` and `nin` take no key predicate, since they cannot use an index today either. `is_null` and `is_not_null` take none either. A first draft asked the key column as well, and the checks under "Checks on the implementation" showed that it changed no plan on most engines and made `is_not_null` slower on MySQL 8.0.13.

Three things must hold or the lookup returns wrong rows without an error. The search value is cast to the page's collation before its weight string is taken, because a bound value arrives in the connection's collation and its weight string under that collation matches nothing stored (in a probe run of 2026-10-06 on MySQL 8.0.46, 58 of 62 lookups returned the wrong rows without the cast and none with it). The collation name comes from the page, never from a constant, and is checked against `[A-Za-z0-9_]+` before it reaches SQL. And the value is bound once for each place the key text uses it, with positional placeholders, because a named placeholder used twice fails under native prepares (SQLSTATE HY093).

### Which equality starts a read

[`0056`](0056-filtered-default-order-reads-drive-from-a-full-width-equality-leaf.md) starts a filtered default-order read from an equality leaf, and it gains a second pass here. The first pass is unchanged and takes the first `eq` on an integer, numeric or datetime slot that has a recorded index. Only when it finds none does the second pass take the first `eq` on a string slot whose page has a key, and the index hint names that key's index. An integer, numeric or datetime equality therefore starts the read even when a text equality is written before it, and every query that starts from a leaf today keeps its plan.

### How the engine knows a page's shape

It reads the shape from the data dictionary, the server's own description of the page table, and never from a column in the registry. The table is the only fact. A crash between a page's `CREATE TABLE` and the registry transaction can leave an orphan table that a later provisioning reuses by name (`CREATE TABLE IF NOT EXISTS`), so a registry flag could name a shape the table does not have, and the compiler would then fail on a column that is not there. [`0043`](0043-pages-provision-only-indexed-columns.md) rejected persisting `is_indexed` for the same reason, a denormalised copy of a dictionary fact. The head width and the collation are facts of the same kind. Keeping them out of the registry also means no registry migration.

The schema snapshot already records, for every full-width two-part index, which column it covers. A key column qualifies as it stands. When that map holds a column ending in `_k`, one more query reads `information_schema.COLUMNS` for the page tables and takes the collation of each `i_str_NN` and the width of each `i_str_NN_k`, and the head is that width minus 16. A slot with no key compiles to today's SQL. That covers a slot on a prefix page, on a page from before 0043, behind a key index an operator made invisible, behind a key column that is not a `VARBINARY`, and with a collation name that is not a plain identifier. The cost is one statement per cold snapshot refresh for a model that lives on key pages, and nothing on a warm read.

### Capacity and reservation

On a key page `i_str_01` is in no index and its key is. `IndexedSlotPredicate` answers "is this slot indexed" for the reserver and for both capacity readers, so it now accepts the slot column or its `_k` companion. That change has to land before any key page is built. Without it the reserver refuses every string slot on a key page, the Watcher counts none of them as usable and provisions another page on every tick.

### Existing fields and compaction

No existing page is altered, and no operator path moves a field onto a key page. Reservation prefers the oldest page with a free indexed slot, starting with the pages the model already occupies, and compaction ([`0033`](0033-operator-initiated-model-compaction.md)) only targets pages the model already occupies. A deployment that upgrades therefore keeps its string fields on prefix pages, and key pages receive string fields only as the older pages' free string slots run out. A fresh database gets key pages everywhere. Three page shapes now exist for good: the sixty-column pages from before 0043, prefix pages, and key pages.

### What a key page promises, and what it does not

It promises that a text equality, or a list of them in `in`, is answered from an index in `entry_id` order, and that a read started from it touches about two rows for each of the 51 rows a page needs. It promises that a text prefix or range bound narrows the index within the head, which covers the first 32 bytes of the weight string, about 16 Latin letters or 8 CJK characters. And it promises that an equality stays exact for values of any length up to the 4096-character bound.

It does not promise a sort. After the head, keys are ordered by digest, so key order is value order only within the head, and nothing may add an `ORDER BY` on a key. Field sorts on a string keep reading the slot, as [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) left them. It promises nothing new for `neq`, `nin` or a filter with `or` or `not`, which keep today's SQL, and it changes no page built before it.

## Consequences

**Positive:**

- A text equality that starts a read is bounded by about a page on every engine. At 5% of a tenant it went from 10,009 rows touched on MySQL 8.0.46 and 15,065 on 8.0.13 to 102, with none sorted, and the median time was 0.03 of today's on both MySQL versions and about 0.5 on MariaDB (runs of 2026-10-18).
- A row with many long values can be updated and deleted. In every undo scenario above the key page succeeded for all 25 slots, on both MySQL versions measured and on MariaDB 10.11 (runs of 2026-10-19).
- A text equality is exact for values of any length, and the value missed by the prefix index is found.
- An 8 KB InnoDB page size builds a page again. Today's page fails there with errno 1071 and the key page builds on all four engines. A 4 KB page size still builds neither shape (errno 1071 for today's page and 1118 for the key page), so the refusal in `bootstrap()` stays a separate item.

**Negative:**

- A text prefix or range bound that reaches past the head, on a column where every value shares that head, can no longer narrow the index, and this was accepted together with the choice of key. A lower bound 1% from the top of such a column read 198,100 rows in 1.6 to 2.2 s on MySQL 8.0.46 (200,000 rows, URL-like values of 20 to 148 characters whose first 12 to 140 characters are the same in every row), against 2,001 to 2,003 rows in 8 to 19 ms on today's page (runs of 2026-10-18 and 2026-10-19). A prefix that stayed inside the head ran within 1.06 to 1.30 times today's time on the same data.
- Writes cost more. These are the ratios of the key page's time to today's page's time for 1,000 rows with 255-character values in all 25 slots, the median of 7 paired repetitions on 10,000 preloaded rows (run of 2026-10-19). Each repetition varied widely (for MySQL 8.0.46, the insert ran from 0.44 to 9.03 times), so only the order of magnitude holds. These are SQL level figures. "Checks on the implementation" has the engine's own write path timed in paired runs on short values, where the key page cost little, and the one unpaired load that was timed earlier.

  | Operation | MySQL 8.0.13 | MySQL 8.0.46 | MariaDB 10.11 | MariaDB 11.8 |
  | :-- | :-- | :-- | :-- | :-- |
  | insert | 1.01 | 2.88 | 1.73 | 1.74 |
  | update | 0.96 | 2.21 | 1.30 | 1.14 |
  | null out one slot | 1.65 | 0.91 | 4.46 | 1.96 |
  | cascade delete | 0.69 | 0.73 | 0.42 | 0.40 |

- Rows are bigger. At 400,000 rows with four populated string slots averaging 309 bytes per row, table data was 1.54 times today's and index data 0.98 times, and point reads took 1.25 to 1.40 times as long with a buffer pool of 256 MB (run of 2026-10-19). A row with all 25 string slots filled stays in the page up to about 272 ASCII characters per slot against about 320 today, because each key adds 50 bytes. In the null-out runs, 255 characters per slot gave the same table size on both shapes and 300 characters made the key page's table larger.
- A process from an earlier release cannot claim a string slot on a key page, and its Watcher may provision extra pages, because its idea of an indexed slot does not know the key. Its reads on a key page are correct and unindexed, and its writes are fine. The rollout must finish before the first key page is built, which `docs/deployment.md` says.
- A deployment keeps three page shapes for good, and an upgraded one keeps its string fields on prefix pages for a long time. Both cost reasoning in the compiler and the reserver, and neither is hidden from an operator.
- A cold snapshot refresh for a model on key pages issues one more dictionary query.

**Rejected alternatives:**

- Keep the prefix index and document the limits. This leaves all three problems, including the missed value, which is a wrong answer and not only a slow one.
- Add the key as a second index beside the prefix index. A full page already carries 62 of the 64 keys the server allows, so 25 more cannot fit, and the prefix index is the source of the undo failures.
- A plain weight key, the weight string cut to a fixed width with no digest. It cannot tell apart two values that share their first bytes, so an equality on them reads every row that shares the start and cannot stop after a page, and a width that covers 4096 characters does not fit in an index key. Widening it to 256 bytes made a 1,000-row null-out 9.58 times as slow as today's page on 255-character values and grew the table from 55.6 MB to 2,352.6 MB (MySQL 8.0.13, run of 2026-10-19).
- A digest alone, with no head. It makes equality exact and gives up every prefix and range bound, which the prefix index serves today for the first 766 characters.
- A longer head. It would shrink the prefix and range limit above and cost the row size the null-out runs show. The compiler reads the head from each page, so a later release can change it without breaking old pages, and a `Config` parameter can be appended then. No setting ships now.
- A registry flag for the page shape. Rejected under "How the engine knows a page's shape".
- Altering existing pages. [`0012`](0012-immutable-extension-page-ddl.md) forbids it on populated pages.

## Checks on the implementation

These runs of 2026-10-28 used the implementation as built, through the engine's own read path. The data was four string fields on one model, 100,000 rows for each of two tenants, read 50 rows to a page. `label` has a value held by 5% of the rows and another held by 50%. `tier` is two lowercase letters and a number. `url` is a 44 character start shared by every row, then eight digits. `note` is filled on 5% of the rows. Rows touched come from the server's `Handler_read_*` counters. Times are the median of 7 runs of the same statement on a warm server, so they show the size of a difference and not a precise ratio. The engines were MySQL 8.0.13, MySQL 8.0.46, MariaDB 10.11.19 and MariaDB 11.8.9 for the key page, and MySQL 8.0.13 and 8.0.46 for today's prefix page.

On the key page a text equality at 5% and at 50% touched 102 rows and sorted none, in ascending and descending order and on the first and the tenth page, on all four engines. A `label` equality together with a prefix on `tier` touched 515 and 551 rows with none sorted, against 6,003 rows and 51 sorted on the prefix page of MySQL 8.0.13, and 5,460 and 51 on MySQL 8.0.46.

| Read, first page | Key page, MySQL 8.0.13 | Prefix page, MySQL 8.0.13 | Key page, MySQL 8.0.46 | Prefix page, MySQL 8.0.46 |
| :-- | :-- | :-- | :-- | :-- |
| `label` equal, 5% | 102 touched, none sorted, 1.7 ms | 14,957 touched, 51 sorted, 284 ms | 102 touched, none sorted, 2.7 ms | 9,937 touched, 51 sorted, 27 ms |
| `label` equal, 50% | 102 touched, none sorted, 1.1 ms | 200,107 touched, 51 sorted, 1,935 ms | 102 touched, none sorted, 2.3 ms | 150,028 touched, 51 sorted, 332 ms |

For bounds that stay inside the head, on MySQL 8.0.46 the key page and the prefix page touched the same rows for the bounds that stay inside the head (1,995 for the prefix `ab`, 2,079 for `lt ab`, 3,853 for `between ba and bc`, about 110,000 for the 10% bounds). On MySQL 8.0.13 the key page touched about 1.6 times as many rows (5,038, 5,248 and 9,683 against 3,044, 3,170 and 5,831), with times of 523, 574 and 455 ms against 447, 291 and 626 ms. The cause was not established, and MySQL 8.0.13 is the supported floor. On MariaDB the optimiser read the primary key in `entry_id` order for most of these bounds and sorted nothing.

Bounds past the head show the accepted limit again. A prefix of the shared `url` start plus four digits touched 105,001 rows on MySQL 8.0.46 in 475 ms on the key page, against 10,001 rows in 27 ms on the prefix page, and 110,053 rows in 725 ms against 15,053 rows in 21 ms on MySQL 8.0.13. The late lower bound on `url` touched the same rows on both shapes on 8.0.13 (110,053), and 105,001 against 10,001 on 8.0.46. On MariaDB the key page touched 5,102 rows for the prefix and 95,102 for the late bound, with no prefix page run to compare. The counts differ from those in the Consequences, which came from a different dataset, and the direction is the same.

The null tests take no key predicate. `is_null` at 95% never used the key index on any engine, because the optimiser preferred the tenant index. `is_not_null` at 5% used it only on MySQL. On MySQL 8.0.46 the plan matched the prefix page (10,025 touched, 51 sorted, about 22 ms either way). On MySQL 8.0.13 the key predicate made it worse, with 25,113 rows touched, a sort and 497 ms, against 1,618 rows, no sort and 8.5 ms on the prefix page. MariaDB read the primary key in order. The key predicate for both tests was therefore removed.

| Other check | MySQL 8.0.13 | MariaDB 10.11.19 |
| :-- | :-- | :-- |
| Cold snapshot refresh, median of 300 loads | 5.92 ms on a key page, 4.18 ms on a prefix page | 137.26 ms on a key page, 92.62 ms on a prefix page |
| Liberator sweep of one slot with 25 string slots filled, 5,000 rows, chunks of 500, median of 3 rounds, 30 character values | 2,296 ms on a key page, 839 ms on a prefix page | 2,100 ms on a key page, 1,784 ms on a prefix page |
| The same sweep with 255 character values | 4,157 ms on a key page, 7,007 ms on a prefix page | 2,671 ms on a key page, 2,834 ms on a prefix page |
| Promotion backfill of a string field, 30,000 rows, 61 ticks, median of 3 rounds | 5,961 ms on a key page, 2,716 ms on a prefix page | 27,432 ms on a key page, 28,182 ms on a prefix page |

The MySQL 8.0.13 timings varied a lot between rounds. The sweep on the prefix page with 255 character values took 13,770, 3,551 and 7,007 ms, and the backfill on the key page took 8,965, 5,961 and 3,178 ms, so the medians are weak evidence for the direction. The refresh is one more dictionary read, and it happens once for each schema change in a process.

For the probe's data a page table was 26.6 MB of data and 39.8 MB of index on the key page, against 10.5 MB and 22.6 MB on the prefix page, the same on both MySQL versions. The load of both tenants through the engine's own write path took 254 s on the key page and 133 s on the prefix page on MySQL 8.0.13, and 204 s and 191 s on 8.0.46. Each is one unpaired run, so this shows the order of magnitude only, and the paired runs below did not reproduce the gap.

The engine's own write path was then timed in paired runs on 2026-10-30. They used the comparative benchmark's product model (eight fields, three of them filterable strings of 6 to 10 characters, five tenants), 100,000 rows loaded through the public bulk write, five interleaved rounds against a generated-column table on the same server, and a container with 2 CPUs and a 1 GB buffer pool. The tree before this record was compared with the branch tip, which differ on the write path only by the key. On MySQL 8.0.46, one run each, the load took 24.1 s before and 21.9 s after. A 1,000-row batch took 160 ms before and 195 ms after, with per-round ratios to the generated-column table from 1.96 to 4.69 before and from 0.41 to 6.99 after, so the two are level. Making a field filterable took 6.7 s before and 8.8 s after. On MySQL 8.0.13, in the order before, after, after, before, the load took 12.1, 14.2, 14.8 and 13.8 s and a 1,000-row batch took 169, 181, 188 and 190 ms. The tables grew from 175.6 MB to 206.8 MB on both versions. On this data the key page cost about 12% more load time on MySQL 8.0.13, the mean of two runs each, and no more on 8.0.46. The earlier load differed in its data (four string fields, one of them a 52 character value), in its server (an installation on the development machine) and in its size (200,000 rows). These rows also do not hold long values in many string slots, which is the case the SQL level ratios in the Consequences describe.

On correctness, the same read path returned for each of 784 probes over values chosen to confuse a lookup the same entries as the slot column compared alone, on four connection settings. Breaking one part of the layout at a time showed what each part carries. Without the cast to the page collation 546 and 548 probes were wrong, on the two settings whose collation differs from the page's. Without the zero padding of lower bounds 123 were wrong, without the 0xFF fill of upper bounds 105, and with a prefix upper bound built from the padded head 215. MariaDB 11.8 passed the 682 tests selected by name for retype, ticks, the daemons, compaction, slots, writes, reads, search and pages, with one MySQL only test skipped.

## Related

- [`0030`](0030-string-slot-storage-type-and-index-prefix.md) chose `TEXT` and the 766-character prefix index. The column type stays and the index changes for pages built after this record.
- [`0056`](0056-filtered-default-order-reads-drive-from-a-full-width-equality-leaf.md) left text equality open. This record answers it and extends the driving rule. Both it and [`0043`](0043-pages-provision-only-indexed-columns.md) are still `Proposed`.
- [`0012`](0012-immutable-extension-page-ddl.md) makes the change forward-only. [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) keeps field sorts on the slot. [`0033`](0033-operator-initiated-model-compaction.md) is why existing fields do not move. [`0054`](0054-mariadb-10-11-support-floor.md) and [`0055`](0055-target-engine-is-detected-not-declared.md) give the two collations and the engine detection the page collation relies on.
