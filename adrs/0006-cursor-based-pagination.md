# 0006 - Cursor-Based Pagination Over Offset Pagination

**Status:** Proposed
**Created:** 2026-04-15

## Context

The engine's read function must serve result sets of unpredictable and potentially enormous size without imposing unbounded query cost on the database. The two dominant alternatives to cursor-based pagination both fail this requirement in ways that scale with data volume rather than page size.

Offset-based pagination (`OFFSET N LIMIT M`) degrades non-linearly as pages deepen. MySQL must scan and discard the first N rows before returning the requested page, which means that page 1,000 of a large result set is as expensive as a near-full table scan — and the cost grows every time new data is ingested. This behavior is invisible during development on small datasets and catastrophic in production on large tenants.

`COUNT(*)` shares the same problem. Evaluating the total number of rows matching a filtered predicate requires MySQL to read the entire matching set. On a large tenant, this can take seconds and contends with the InnoDB read path — not because the query is malformed, but because the dataset simply grew. The bounded execution guarantee established in ADR 0005 (Two-Query Bounded Read Path) requires that every database operation touches at most a fixed, known number of rows; `OFFSET` and `COUNT(*)` both break this invariant, and no index structure eliminates their worst-case cost.

The practical consequence is non-obvious for downstream callers: many UI grid patterns — page numbers, jump-to-page navigation, total record counts — depend on offset semantics or on a total row count, neither of which the MySQL Native Driver supports. Those expectations are reasonable but cannot be served by the engine. Documenting the rejection explicitly is necessary because callers will encounter this boundary and need to understand it was deliberate.

## Decision

The engine's read function uses cursor-based (keyset) pagination exclusively. The function does not accept `OFFSET` parameters and does not return a total row count. The MySQL Native Driver will not execute `COUNT(*)` against filtered result sets under any circumstances.

Callers navigate forward through a result set by supplying an opaque cursor value derived from the `id` of the last record seen. The function returns the next-page cursor as part of its return value when a subsequent page exists; absence of a next-page cursor signals end-of-results. This design matches the Two-Query Bounded Read Path (ADR 0005): the Paginated Probe uses `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`, touching exactly `page_size + 1` rows regardless of result set depth or total dataset size.

Jump-to-page navigation, "Page N of M" displays, and deep-linked page URLs are explicitly out of scope for the MySQL Native Driver. Callers requiring these capabilities must inject an alternative driver via the Driver/Adapter interface defined in the architecture blueprint (§3). A search-engine-backed driver such as a `MeilisearchDriver` can maintain its own document index with efficient total-count and offset semantics, supporting consumer patterns that need them without placing those costs on MySQL. The core ingestion engine remains unaffected; the driver boundary absorbs the difference in pagination model.

## Consequences

**Positive:**

- Query execution cost is O(page_size) at every depth — page 1 and page 10,000 produce identical query plans and touch identical numbers of rows.
- Callers are never silently punished as their data grows. A filtered query returning 50 rows today and 50 million rows next year places the same load on MySQL per page retrieved.
- Eliminates a class of production incidents caused by deep-offset queries on unexpectedly large tenants that performed acceptably during development.
- Pagination behavior is consistent with the Two-Query Bounded Read Path (ADR 0005) and requires no special-casing at the query executor level.

**Negative:**

- Jump-to-page patterns cannot be served by the MySQL Native Driver. Callers expecting these patterns must either present a next/previous or infinite-scroll model to their consumers, or inject an alternative driver.
- Total row counts are unavailable. Callers wishing to surface record counts to their consumers must derive them through a different mechanism (an alternative driver, separately-aggregated count, etc.) or omit the affordance.
- Deep linking to an arbitrary page is not possible. Sequential navigation forward from the beginning of the result set is the only supported access pattern.
- A cursor is invalidated if the caller changes sort order or filter parameters mid-session. Any such change requires restarting pagination from the beginning of the new result set.
