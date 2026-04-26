# 0014 - Schema-Level Safety Over Runtime Circuit Breaking

**Status:** Proposed
**Created:** 2026-04-18

## Context

Database-backed APIs commonly protect themselves from expensive queries through runtime circuit breakers: mechanisms that monitor scanned row counts, elapsed time, or memory consumption during query execution and abort the query if a threshold is exceeded. These approaches treat query safety as a runtime concern — the system accepts any query, begins executing it, and intervenes if execution exceeds acceptable bounds.

StarDust's MySQL Native Driver takes a fundamentally different position. The driver already enforces pre-flight rejection of unindexed filters (ADR `0004`), bounded read paths (ADR `0005`), and cursor-based pagination (ADR `0006`). These mechanisms collectively ensure that every query reaching MySQL is structurally bounded. The remaining question is what happens when a query passes all structural checks but still performs poorly — for example, filtering on a low-cardinality indexed column (a boolean with 50/50 distribution) where the index exists but provides minimal selectivity.

A runtime circuit breaker would catch these cases by aborting the query after scanning N rows. However, this creates unpredictable API behavior: the same query with the same parameters succeeds or fails depending on data distribution, which changes over time as data is ingested. Consumers cannot predict whether their query will complete, and the failure mode — a mid-execution abort — is indistinguishable from a system error without careful error categorization.

## Decision

The MySQL Native Driver does not implement runtime scanned-row tracking, query-time circuit breakers, or dynamic abort mechanisms. Query safety is enforced entirely at the schema level through pre-flight validation, not at execution time through runtime monitoring.

The driver relies on three structural guarantees to bound query execution:

1. **Pre-flight rejection** (ADR `0004`): Queries referencing fields without `is_filterable = true` are rejected with `400 Bad Request` before the database is touched.
2. **Bounded page size** (ADR `0005`): The paginated probe touches at most `page_size + 1` rows. The bounded fetch touches at most `page_size` rows.
3. **Cursor pagination** (ADR `0006`): No `OFFSET` or `COUNT(*)` — every page retrieval is O(page_size).

A slow index scan caused by a low-selectivity filter is treated as an architectural configuration failure — the schema designer provisioned an index on a field that does not provide sufficient selectivity for the expected data distribution. The framework does not attempt to detect or compensate for this condition at runtime. The responsibility for ensuring high index cardinality belongs to the schema designer, not the query executor.

## Consequences

**Positive:**

- API behavior is deterministic. A query that succeeds today will succeed tomorrow regardless of data growth or distribution changes. Consumers never encounter "query worked yesterday but fails today" scenarios caused by shifting row counts.
- No false positives. Legitimate queries that happen to touch many rows (e.g., a filtered scan that correctly matches a large fraction of the dataset within the page-size bound) are never aborted prematurely.
- The query executor is simpler. No instrumentation, row counting, or abort logic is needed in the hot execution path.
- Schema design quality is surfaced as an explicit concern rather than silently compensated for by runtime mechanisms.

**Negative:**

- Low-cardinality index scans are not caught by the framework. A boolean filter on a column with 50/50 distribution will perform a covering index scan touching half the matching rows to find `page_size + 1` results — slow but not aborted.
- The system does not self-protect against poorly designed schemas. If a schema designer provisions an index on a low-selectivity field, the resulting performance degradation is not detected or reported by the framework.
- Operators must rely on external monitoring (slow query logs, database performance dashboards) to detect low-selectivity index scans rather than receiving framework-level alerts.

**Rejected alternatives:**

- Runtime scanned-row circuit breaker (abort after N rows) — creates unpredictable API behavior where identical queries succeed or fail based on data distribution. Punishes legitimate queries that happen to touch many rows within the bounded page-size constraint.
- `EXPLAIN`-based query validation before execution — adds latency to every query. MySQL optimizer estimates are unreliable, especially for multi-table joins and complex predicates. The approach is fragile across MySQL versions and provides false confidence.
- Query timeout enforcement (`MAX_EXECUTION_TIME`) — a blunt instrument that does not distinguish between a slow query caused by a legitimate large result set and a slow query caused by a missing index. Timeouts create the same unpredictable success/failure behavior as row-count circuit breakers.
