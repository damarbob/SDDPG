# 0001 - Extension Tables Over EAV

**Status:** Accepted
**Created:** 2026-04-12

## Context

StarDust needs to store flexible, model-driven payloads while preserving high-throughput ingestion and predictable retrieval. A naive Entity-Attribute-Value (EAV) design, a giant sparse table, or a JSON-only approach would each optimize one concern while materially harming another: EAV increases join complexity and weakens type safety, sparse tables become unwieldy as models evolve, and JSON-only storage encourages unbounded filtering behavior. The system needs a storage pattern that keeps the full payload intact while exposing a controlled set of typed, queryable fields.

## Decision

We will use a vertically partitioned schema built around a core payload table (`entry_data`) plus 1:1 extension tables (`entry_slots_page_X`). The `entry_data` table remains the authoritative record for the full entry payload and lifecycle metadata, while extension tables hold only the explicitly mapped, typed slot columns used for bounded retrieval and filtering.

This choice intentionally avoids an EAV schema. Each extension page uses fixed typed columns for strings, integers, numerics, and datetimes, keyed by `entry_id`, and is treated as a deterministic projection of the authoritative payload. New capacity is created by provisioning additional extension pages rather than mutating existing populated pages or introducing unconstrained attribute rows.

## Consequences

**Positive:**

- Queryable fields stay strongly typed and indexable, making bounded reads and predictable query planning practical.
- The full payload still exists in `entry_data.fields`, so the system preserves flexibility without forcing every field into first-class indexed storage.
- Storage behavior is explicit and auditable: developers can reason about which fields are queryable and where they live physically.
- Page-based growth is operationally safer than reshaping populated tables because capacity expands by adding new extension pages.

**Negative:**

- The physical schema is more complex than a single JSON table and requires page provisioning, registry management, and slot assignment logic.
- Reads may need joins across the core table and one or more extension pages, so the execution strategy must stay disciplined.
- Queryability is limited to fields that have been deliberately mapped into slots; purely ad hoc filtering is not supported by the default architecture.
