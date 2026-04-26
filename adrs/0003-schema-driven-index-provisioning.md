# 0003 - Schema-Driven Index Provisioning

**Status:** Proposed
**Created:** 2026-04-12

## Context

StarDust supports dynamic, model-defined fields, but the system cannot safely index every possible slot by default. Over-indexing would increase write amplification, storage cost, and schema churn, while deciding whether to index a field only when queries begin arriving would make performance unpredictable and encourage runtime exceptions as a normal control path. The project needs a deterministic rule for which fields become filterable and how that choice is materialized in storage.

## Decision

Index provisioning will be schema-driven and decided at model-field registration time through explicit metadata, primarily the `is_filterable` flag. When a field is marked filterable, the corresponding slot column is provisioned with a composite B-tree index that includes `tenant_id` and the slot column. When a field is not marked filterable, the slot may still exist for typed retrieval, but it is not considered valid for filtering.

This decision makes the schema registry the source of truth for queryability. Indexes are incorporated into extension page DDL when pages are provisioned rather than added opportunistically in response to live query behavior.

## Consequences

**Positive:**

- Query performance becomes a deliberate schema concern instead of a runtime gamble.
- The system can reject unsupported queries consistently because "filterable" is an explicit design choice, not an inferred property.
- Capacity planning and write costs remain more predictable because indexing is controlled and auditable.
- Teams gain a clear operational workflow: if a field must be filterable, the schema must say so first.

**Negative:**

- Enabling new filters is not instantaneous; it requires schema-level planning and provisioning rather than an ad hoc API change.
- Incorrect metadata decisions can under-provision or over-provision indexes until the schema is revised.
- The system becomes less permissive for exploratory querying because queryability is intentionally gated by design-time choices.
