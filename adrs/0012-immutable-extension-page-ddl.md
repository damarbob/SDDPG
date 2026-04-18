# 0012 - Immutable Extension Page DDL (Empty-Table-Only Schema Changes)

**Status:** Accepted
**Created:** 2026-04-18

## Context

StarDust's extension tables (`entry_slots_page_X`) are created by the Watcher daemon when global slot capacity drops below its provisioning threshold. Each page is provisioned with a fixed set of typed slot columns and composite B-tree indexes determined at creation time by the schema registry's `is_filterable` flags.

In traditional relational systems, evolving a table's schema — adding columns, changing types, or adding indexes — is accomplished via `ALTER TABLE`. On MySQL/InnoDB, `ALTER TABLE` on a populated table acquires metadata locks that block all concurrent reads and writes for the duration of the DDL operation, even when using `ALGORITHM=INPLACE`. On a hot extension table receiving concurrent ingestion and query traffic, a single `ALTER TABLE` can stall the entire tenant's throughput for seconds or minutes depending on table size.

The zero-dependency core (ADR `0002`) rules out external schema migration tools (`pt-online-schema-change`, `gh-ost`) that work around metadata lock contention through shadow-table strategies. These tools introduce operational complexity and an external tooling dependency that the architecture explicitly avoids.

## Decision

`ALTER TABLE` on populated extension pages is strictly forbidden. Schema changes to the extension layer are accomplished exclusively by provisioning new extension pages. A new page is created as an empty table with the desired column layout and indexes, and new field-to-slot mappings are assigned to it. Existing populated pages are never altered.

This means the column layout and index composition of an extension page are frozen at provisioning time. If a field's indexing requirements change after its slot has been populated, the change is handled through the eviction lifecycle (ADR `0009`): the old slot is tombstoned and swept, and the field is re-provisioned on a new or different page with the correct index configuration.

## Consequences

**Positive:**

- Metadata lock contention on live tables is eliminated. `CREATE TABLE` on a new, empty table acquires no locks on existing tables, so provisioning never blocks concurrent reads or writes on populated pages.
- DDL operations are inherently safe to execute under high-throughput ingestion. The Watcher can provision new pages on its leisurely polling schedule without creating write backpressure.
- The provisioning path is simple and deterministic: create table, create indexes, update registry. There is no need for online DDL algorithms, shadow-table swaps, or version-specific MySQL behavior.
- Rollback is trivial: a failed `CREATE TABLE` simply means the table does not exist. No partially altered live table is left in an inconsistent state.

**Negative:**

- Index composition is frozen at provisioning time. Retroactive schema changes to existing pages — such as adding an index to a slot that was originally provisioned without one — require data migration through the tombstone-sweep-reprovision cycle, not a simple `ALTER TABLE ADD INDEX`.
- The number of extension pages grows monotonically. Pages are never consolidated or merged, which increases the join surface for cross-page queries over time.
- Fields that change their `is_filterable` status frequently incur repeated eviction and re-provisioning overhead rather than a single in-place index change.

**Rejected alternatives:**

- Online DDL (`ALTER TABLE ... ALGORITHM=INPLACE`) — still acquires metadata locks during the prepare and commit phases. Behavior varies across MySQL versions and storage engine configurations, making it unreliable as a general-purpose guarantee under high concurrency.
- `pt-online-schema-change` or `gh-ost` — introduces external tooling dependencies that violate the zero-dependency core (ADR `0002`). Both tools add significant operational complexity (trigger-based or binlog-based replication of changes to a shadow table) for a system designed to avoid external infrastructure.
- Shadow table with atomic rename swap — requires careful coordination of foreign key relationships, briefly acquires table-level locks during the rename, and produces a moment of unavailability for the affected page. The complexity is disproportionate to the benefit when new-page provisioning achieves the same outcome without touching live tables.
