# 0023 - Minimum Supported MySQL Version: 8.0.13

**Status:** Proposed
**Created:** 2026-04-23

## Context

The schema registry's atomicity invariants (ADR `0017`) rely on a partial unique index `UNIQUE (field_id) WHERE status IN ('assigned', 'backfilling', 'ready')` on `stardust_slot_assignments`. This constraint guarantees a field has at most one live slot at any time and is the database-level enforcement that makes the closed slot-status state machine correct under concurrent daemon writes.

MySQL natively supports functional / conditional unique indexes from 8.0.13 onward. The Schema Reference (§4.4) currently documents a generated-column workaround for earlier versions: a stored generated column containing `field_id` when `status` is live and `NULL` otherwise, with a standard `UNIQUE` on that column.

The architecture has not declared a minimum supported MySQL version. Without one, the workaround must be treated as a first-class supported configuration, doubling the per-feature implementation surface (every registry-mutating code path needs a "did the workaround column update?" assertion in addition to the native-index check). The Glossary, Blueprint, and Schema Reference are silent on the version policy.

## Decision

**Minimum supported MySQL version: 8.0.13.**

The generated-column workaround documented in [`schemas/schema_reference.md`](../schemas/schema_reference.md) §4.4 is removed. Implementations may not target MySQL versions older than 8.0.13.

### Why 8.0.13 specifically

8.0.13 is the earliest version that natively supports:

- Functional / conditional unique indexes (the registry constraint).
- `JSON_TABLE` (used in legacy-data-migration tooling — out of scope here, but the version requirement aligns).
- `WITH ... AS (...)` common table expressions (used in registry diagnostic queries).
- `EXPLAIN ANALYZE` (used in operator runbooks for the cardinality advisory introduced by ADR `0019`).

Picking 8.0.13 captures the entire feature set the architecture already assumes without imposing a more recent floor than necessary.

### Out-of-scope versions

- **MySQL 5.7 and earlier**: explicitly unsupported. The partial-unique workaround is removed; ADR `0011`'s SAVEPOINT semantics are weaker; `JSON_TABLE` is unavailable.
- **MariaDB**: not supported as a drop-in replacement. MariaDB's JSON storage representation, partial-index syntax, and `SKIP LOCKED` semantics differ enough that targeting it would be a separate ADR with its own compatibility matrix.
- **Percona Server**: supported because it is wire- and feature-compatible with the corresponding upstream MySQL version. Operators running Percona 8.0.13+ are inside the supported envelope.

### Documentation Updates

The following documents MUST be updated as part of this ADR landing:

1. [`schemas/schema_reference.md`](../schemas/schema_reference.md) §4.4 — remove the generated-column workaround note and replace with a flat statement of the 8.0.13 requirement.
2. [`README.md`](../README.md) at the repo root — add a "Requirements" section if not present, listing the MySQL version floor.
3. The Architecture Blueprint — add a brief "Operating environment" sub-section under Section 1 noting the version requirement and pointing at this ADR.

### What changes for existing deployments

Nothing. ADRs in the project are still in flux (this ADR, like all current ADRs, is in `Proposed` status), and there is no production deployment yet. The minimum-version decision is being made before deployment, not as a breaking change to existing operators.

## Consequences

**Positive:**

- The registry's atomicity invariants reduce to native MySQL constraints. Implementation reviewers can read the `stardust_slot_assignments` DDL once and trust the constraint, rather than verifying that every code path also maintains a generated-column shadow.
- Operator runbooks can assume modern MySQL surface area: `JSON_TABLE`, CTEs, `EXPLAIN ANALYZE`, `SKIP LOCKED`. Diagnostic queries do not need legacy-version fallbacks.
- The compatibility matrix is simple: "MySQL 8.0.13+ or Percona 8.0.13+." Documentation, support, and bug-triage are bounded.

**Negative:**

- Operators on MySQL 5.7 (still common in older enterprise estates) cannot deploy StarDust without first upgrading. The upgrade path is well-documented upstream but is not zero-effort.
- MariaDB users are excluded from the supported set even though MariaDB is wire-compatible for many SQL workloads. A future MariaDB-targeted ADR could change that, but this ADR does not promise it.
- The generated-column workaround removed from the Schema Reference takes with it the implicit "we tried to support older MySQL." Operators reading older drafts of the Schema Reference may expect that fallback path to remain.

**Rejected alternatives:**

- **Keep the generated-column workaround as a supported fallback** — doubles the registry-mutation code paths and forces every implementation to maintain two equivalent constraint mechanisms. The workaround's correctness is harder to audit than the native index, and every defect that would land in one path silently lands in both.
- **Pick MySQL 8.0.16** (full functional-index support, including expressions on multiple columns) — gains nothing over 8.0.13 for the constraints actually used by the registry. The minor floor was rejected to keep the version requirement as accommodating as possible.
- **Pick MySQL 8.4 LTS** (the current LTS at this writing) — too aggressive a floor. The architecture uses no 8.4-only features and forcing operators onto LTS for no functional reason narrows the deployment envelope without benefit.
- **No minimum version, document features used and let operators verify** — the de facto current state. Rejected because it pushes the compatibility-matrix work onto every deployment, every time, and makes the project's testing matrix indeterminate.

## Related

- ADR `0002` — MySQL Native, Zero-Dependency Core
- ADR `0011` — Chunked Bulk Ingestion (uses SAVEPOINT semantics, present in 8.0.13)
- ADR `0017` — Schema Registry as Coordination Contract (relies on the partial unique index)
- [`schemas/schema_reference.md`](../schemas/schema_reference.md) §4.4
