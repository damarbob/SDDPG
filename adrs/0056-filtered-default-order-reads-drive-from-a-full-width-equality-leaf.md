# 0056 - Filtered Default-Order Reads Drive From a Full-Width Equality Leaf

**Status:** Proposed
**Created:** 2026-10-26

## Context

[`0005`](0005-two-query-bounded-read-path.md) makes the read a probe that selects only ids with `LIMIT page_size + 1`, followed by a bounded fetch. [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) scoped the probe's bounded-discovery guarantee to the default ordering and the `created_at` sort, on the strength of measurements taken without a filter. A filtered read in the default order is not bounded either.

The probe leads with `entry_data`, orders by `entry_data.id` and joins the slot pages. For a filter on an integer, numeric or datetime slot, MySQL chooses that slot's own index, reads the whole match set and sorts it to find one page. Measured with session handler counters (rows touched and `Sort_rows`, not wall time), on 200 000 rows in two tenants with a page size of 50, an equality on an integer slot matching 5% of tenant 1 (5 080 rows) touched 10 161 rows and sorted 51 on MySQL 8.0.46, and touched 15 293 and sorted 51 on MySQL 8.0.13. MariaDB 11 already stops at 102 rows in every equality case measured, about two reads for each of the 51 rows a page needs, and MariaDB 10.11 does in most of them (the others touch between 160 and 1 155 rows and sort none). At 25% and 50% the MySQL optimiser already picks a good plan (160 to 440 rows), so whatever fixes the low-selectivity case must not disturb those.

The same pattern appears on page 10 as well as page 1, ascending and descending, and whether the equality sits on the first joined page or the second.

## Decision

When the filter is a pure AND (the join strategy), the sort is the default order or `Id`, and at least one leaf is an equality on a full-width slot column, the compiler drives the query from that leaf's page. Full width means an integer, numeric or datetime slot. The leaf must have an index recorded in the schema snapshot, and the first such leaf in filter order is used.

```sql
SELECT STRAIGHT_JOIN entry_data.id
FROM entry_slots_page_N p0 FORCE INDEX (`<index from the snapshot>`)
INNER JOIN entry_data ON entry_data.id = p0.entry_id AND entry_data.tenant_id = p0.tenant_id
INNER JOIN <the other pages, as today> ...
WHERE entry_data.tenant_id = ? AND entry_data.model_id = ? AND p0.tenant_id = ? AND <keyset on p0.entry_id> AND <filter>
ORDER BY p0.entry_id ASC|DESC
LIMIT ?
```

Page aliases are assigned as they are today, so the driving page keeps its alias even when it is not `p0`, and only the order the tables are written in changes. The keyset predicate and the `ORDER BY` move from `entry_data.id` to the driving page's `entry_id`. The `INNER JOIN` forces the two to be equal, so the ordering is the same and the cursor format is unchanged.

Each part of the shape below was measured, except the explicit tenant predicate, which is a precaution (removing it changed no count in 32 pairs on each of MySQL 8.0.46 and MariaDB 11.8).

- **The join order is pinned with `STRAIGHT_JOIN`, driving page first.** The hint on its own is not enough. With `FROM entry_data` still first, a single equality matching 50% of the tenant went from 222 rows touched to 200 001 on MySQL 8.0.46 (300 069 on 8.0.13), with a sort, because the optimiser starts from `entry_data` and sorts the whole join.
- **`FORCE INDEX` names the leaf's own index.** Without it, with a string leaf in the same AND, ordering by the page's `entry_id` touched 13 618 rows against 768 with it (MySQL 8.0.46, 200 000 rows, 915 matches).
- **`p0.tenant_id = ?` is stated explicitly**, so the index range is bounded by both of its key parts.
- **The index name comes from the data dictionary, read into the schema snapshot, and is never built from the `ix_{table}_{column}` convention.** A page from before [`0043`](0043-pages-provision-only-indexed-columns.md), or an index an operator added, may carry any name. A hint naming a missing index fails the read with errno 1176, which turns a slow read into a failing one. An index qualifies only if it has exactly two key parts, `tenant_id` and then the slot column, both at full width, and the optimiser has not been told to skip it. A third key part would sit between the column and `entry_id` and break the order.

Everything else keeps today's SQL byte for byte: the EXISTS strategy, field sorts and `created_at` sorts, string leaves, ranges, `prefix`, `in` and `nin`, and any leaf without a recorded index.

### What does not take the shape

- **`in` and `nin`.** Ordering by the page's `entry_id` took a list matching 50% of the tenant from 238 rows touched to 200 001, and adding the hint left it at 49 936 touched with 49 875 sorted.
- **String equality.** A string slot is indexed on a 766 character prefix, which cannot answer an equality alone. Driving from it with the hint touched 5 037 rows and sorted 4 985, where today's shape touched 9 971 and sorted 51. A string leaf is never chosen as the driving leaf, even when it comes first: in `str5% AND int5%`, driving from the string leaf touched 7 101 rows and sorted 4 990, and driving from the integer leaf touched 3 132 and sorted none.
- **Ranges.** A greater-than matching 5% of the tenant gained nothing from the hint (5 006 rows touched, 4 954 sorted).

## How this narrows 0005

For the filtered default-order read, the probe's bounded-discovery guarantee now holds as follows.

- **One full-width equality:** bounded by about the page size. In all 44 cases measured on each of MySQL 8.0.13, MySQL 8.0.46, MariaDB 10.11 and MariaDB 11 the shape touched 102 rows and sorted none. The cases cover integer, numeric and datetime equality at 5%, 25% and 50% matches, ascending and descending, on page 1 and page 10, driving from the first or second joined page.
- **An equality inside an AND:** bounded by the driving leaf's match count, not by the page size, once the other leaves filter its rows further. On MySQL 8.0.46, two 25% integer equalities on different pages touched 519 rows, against 225 092 for the hint alone. An integer 5% equality with a string 5% equality touched 2 673, against 10 412 today.
- **Still not bounded:** string equality, ranges, `in`, `nin`, `prefix`, and any filter containing OR or NOT, which uses the EXISTS strategy and was not measured.
- **Field sorts** are as [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) left them.

## Open case: text equality

String equality is outside this rule and stays unbounded on MySQL until its schema changes. The prefix index is the obstacle, and removing it is a page-shape change that needs its own decision record. When that lands, this rule extends to it by letting a string leaf drive from whatever full-width key replaces the prefix index.

## Consequences

**Positive:**

- A filtered read on an integer, numeric or datetime equality examines about a page of rows on every supported engine, however large the match set.
- The change is local to the compiler and the snapshot. The cursor format, `PaginatedProbe`, the bounded fetch, `SearchService` and every public signature are unchanged, and `compile_strategy` still reports `joins`.
- MariaDB gains little, since it is already close to a page in the cases measured, and it can lose when a common leaf is written first (MariaDB 11.8, a 5% leaf second, 324 rows touched today against 3 231 driven). **A green MariaDB run therefore proves nothing about the rewrite**, and the tests pin the shape of the SQL as well as the rows touched.

**Negative:**

- One extra query when a schema snapshot is refreshed, reading the page indexes from `information_schema.STATISTICS`. A warm read pays nothing, because the snapshot is cached per schema version.
- Dropping or hiding a page index without bumping `stardust_schema_version` leaves a stale name in the cache, and a read that names it fails with errno 1176 until the next refresh. Nothing in the engine drops a page index, so only a hand-run `ALTER TABLE` can cause it.
- The driving leaf is the first qualifying leaf in filter order, not the most selective one, because the compiler does not know selectivity. In an AND the cost follows that leaf's match count. Every qualifying leaf tried was bounded, but a broader leaf written first costs more than a narrower one written second. In a measured case, a leaf matching 50 008 entries written before one matching 4 touched 150 025 rows on MySQL 8.0.46 and on MariaDB 11.8, against 25 and 31 for today's query. With the rare leaf first it touched 31.
- The MySQL 8.0.13 floor was measured for integer, numeric and datetime equality and behaved like 8.0.46. Text equality alone has no 8.0.13 measurement. String-led AND filters were measured there, and driving from the string touched 3 165 rows and sorted none, against 7 101 and 4 990 sorted on 8.0.46. The rule never drives from a string, so the decision is unchanged.
