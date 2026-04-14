# 0005 - Two-Query Bounded Read Path

**Status:** Accepted
**Created:** 2026-04-14

## Context

StarDust queries may span the core payload table and multiple extension pages. Executing those reads as a single large joined query risks unbounded intermediate work, temporary table creation, and disk spillage, especially as result sets grow over time. The system needs a deterministic read strategy that preserves pagination semantics without allowing synchronous requests to expand into uncontrolled database work.

## Decision

Synchronous reads will use a strict two-query execution pattern. The first query is a paginated probe that selects only entry IDs using covering indexes, cursor constraints, and `LIMIT page_size + 1` to determine whether another page exists. The second query performs a bounded fetch of the full rows for only that small ID set, including any required joins to extension tables.

This pattern is mandatory for the native read path. Cross-page queries and complex filters must not be executed as a single unbounded join intended to both discover and materialize the full result page in one step.

## Consequences

**Positive:**

- Query execution stays bounded by page size instead of total matched rows, which protects memory and reduces spill risk.
- Cursor pagination remains stable as datasets grow because the system only probes the next slice of IDs.
- Full-row materialization is limited to a known, small set of entries, making join costs easier to reason about.
- The read path becomes operationally predictable enough to serve as the baseline for both synchronous listing and export pagination.

**Negative:**

- The application-layer query flow is more complex than a single SQL statement and requires disciplined executor logic.
- Developers must maintain consistency between the probe query and fetch query so they represent the same logical filter.
- Some database-level optimizations available to a single handcrafted query are intentionally left unused in exchange for safety and predictability.
