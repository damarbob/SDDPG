# 0022 - Search Driver Capability Jurisdiction and `is_filterable` Bypass

**Status:** Proposed
**Created:** 2026-04-23

## Context

ADR `0021` establishes that `EntrySearchInterface` accepts a structured `QueryFilter` value object and that the API validates field references and operator support before invoking the driver. ADR `0004` mandates that filters on fields with `is_filterable = false` are rejected with a typed exception at the function-API boundary, before any database interaction.

These two rules collide cleanly in the MySQL Native Driver: `is_filterable` is the registry's signal that a `(tenant_id, slot_column)` index exists, and the pre-flight rejection prevents queries that would force the optimizer into a full scan or a JSON_EXTRACT-in-WHERE.

For an external driver (Meilisearch, OpenSearch, Elasticsearch), the picture is different. These engines maintain their own field-level indexes derived from the JSON payload, independent of MySQL's slot indexes. A field that is `is_filterable = false` in the StarDust registry — and therefore has no MySQL B-tree — may nonetheless be indexed and queryable in the external engine. Strict enforcement of the registry's `is_filterable` flag would artificially cripple external drivers, forcing them to abide by relational-storage constraints that don't apply to them.

The Search Driver Adapter blueprint left this open: should `is_filterable` enforcement bypass when an external driver is active? If the answer is "yes, dynamically," `is_filterable` loses its status as the architectural source of truth. If the answer is "no, always enforce," external drivers are crippled. There is a third path: make the registry's `is_filterable` flag the input to a per-driver capability negotiation, with each driver declaring which fields it considers queryable.

## Decision

The schema registry's `is_filterable` flag retains exclusive jurisdiction over the **MySQL Native Driver's** read path. For non-MySQL drivers, filterability is determined by a **per-driver `supportsFilterOn(field_id) → bool` capability check**, NOT by the registry flag. The driver itself owns the answer.

### Capability Surface (v1)

`EntrySearchInterface` is extended with a closed capability surface:

| Method                                 | Purpose                                                                                                                                                    |
| :------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `supportedOperators(): array`          | Returns the leaf operators (subset of ADR `0021`'s closed set, plus any extension nodes) the driver can execute. Static per driver build.                  |
| `supportsFilterOn(int $fieldId): bool` | Returns whether the driver can satisfy a filter against the given field — covers `is_filterable` for MySQL and arbitrary index state for external engines. |
| `supportsFuzzySearch(): bool`, etc.    | Boolean capability declarations matching the existing search-adapter blueprint sketch. One method per coarse capability.                                   |
| `consistencyModel(): "strong"          | "eventual"`                                                                                                                                                | Returns the active driver's consistency guarantee. Callers translate this into their own consumer-facing surface (e.g., a response header). |

The `MysqlNativeDriver` implements `supportsFilterOn($fieldId)` by returning the registry's `is_filterable` flag for that field. External drivers implement it by consulting their own index metadata.

### Pre-Flight Flow

```
1. Resolve QueryFilter leaf field_name → field_id via schema registry (ADR 0021).
2. For each leaf:
     2a. Check operator ∈ driver.supportedOperators() — reject if not.
     2b. Check driver.supportsFilterOn(field_id) === true — reject if not.
3. Hand the validated QueryFilter to driver.list(...).
```

This pre-flight is uniform across drivers. The MySQL driver effectively re-implements ADR `0004`'s `is_filterable = false → 400` behavior by returning `false` from `supportsFilterOn` for non-filterable fields. External drivers may return `true` for the same field if their index covers it.

### What `is_filterable` Still Does

For all drivers:

- `is_filterable` continues to drive index provisioning on MySQL extension pages (ADR `0003`). It is the slot-reservation eligibility signal (ADR `0016`).
- `is_filterable` continues to be the field-registration policy signal that the schema designer expressed an intent to filter on the field. Operators promoting to an external driver should keep `is_filterable` aligned with intent even when the external driver does not require it for filter acceptance.

For the MySQL Native Driver only:

- `is_filterable` continues to be the literal source of truth for filter acceptance, via the driver's `supportsFilterOn` implementation.

The flag's meaning narrows from "the system will accept filters on this field" to "the schema-designer's intent that this field be filterable." Acceptance becomes a driver-mediated decision.

### Documentation Requirement

The Glossary entry for `is_filterable`, the Architecture Blueprint §2.2, and ADR `0004` MUST cross-reference this ADR so readers understand that `is_filterable`'s enforcement scope is now driver-mediated.

## Consequences

**Positive:**

- External drivers are not artificially crippled. A Meilisearch-backed deployment can accept filters on every JSON field it indexes, regardless of the MySQL registry flag.
- The MySQL Native Driver's behavior is unchanged. ADR `0004`'s pre-flight rejection on `is_filterable = false` continues to work — it is now expressed through the capability surface rather than as a special API-layer check.
- The capability surface is uniform: every driver answers the same questions (`supportedOperators`, `supportsFilterOn`, `consistencyModel`) and the API enforces the same flow. No per-driver pre-flight branching at the API level.
- Operators replacing the MySQL driver with an external engine do not need to re-flag every field — `is_filterable` keeps its design-time meaning, and the driver decides runtime acceptance.

**Negative:**

- The semantics of `is_filterable` shift from "definitive filter-acceptance flag" to "design intent + MySQL-driver-specific acceptance flag." Documentation across the blueprint, glossary, and ADRs must be updated to reflect this — the registry flag's meaning is no longer single-edged.
- A misconfigured external driver could accept filters on fields the schema designer never intended to be queryable (e.g., free-text PII fields). Operators must align driver-level indexing with design intent; the system does not enforce this alignment.
- Capability checks add a per-leaf dispatch on the request hot path. The cost is small (a method call per filter leaf) but real.

**Rejected alternatives:**

- **Strict enforcement always** (`is_filterable = false` rejects regardless of driver) — defeats the entire point of the driver adapter. External engines are designed to remove relational-storage constraints; carrying the constraints into the API layer reimposes them.
- **Driver bypass switch** (a global "drop pre-flight checks if non-MySQL driver active" flag) — too coarse. Loses per-field capability discrimination, allows filters that no driver supports, and makes the API's behavior dependent on a deployment toggle rather than on field-level facts.
- **Make `is_filterable` a per-driver column** (e.g., `is_filterable_mysql`, `is_filterable_meili`) — explodes the registry schema and bakes deployment topology into the data model. The capability-method approach achieves the same effect at runtime without persistent coupling.
- **Have the driver re-declare its supported field set into the registry** (sync external index state into `stardust_fields`) — couples external-engine state mutations into StarDust's coordination contract (ADR `0017`) and forces the system to handle cross-store consistency. The runtime capability check is strictly simpler.

## Related

- ADR `0003` — Schema-driven index provisioning
- ADR `0004` — Fail-fast on unindexed filters
- ADR `0014` — Schema-level safety over runtime circuit breaking
- ADR `0017` — Schema registry as coordination contract
- ADR `0021` — Search driver query representation
- [`blueprints/search_driver_adapter.md`](../blueprints/search_driver_adapter.md) — closes the `is_filterable` jurisdiction question
