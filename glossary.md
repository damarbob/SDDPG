# Glossary — Domain Dictionary

> **Canonical definitions for project-specific terms used across SDDPG.**
> If a term is not defined here, it has no authoritative meaning in this project.
>
> Terms are ordered alphabetically. Each entry includes a definition, optional aliases, and cross-references to related terms or source documents.

---

### Advisory Lock

A MySQL `GET_LOCK()` call used for mutual exclusion. The engine takes two: the Watcher's `stardust_page_provision` lock, held before executing page-provisioning DDL so concurrent provisioning attempts cannot cause table name collisions or metadata lock contention ([`blueprints/watcher_reconciler_daemons.md`](blueprints/watcher_reconciler_daemons.md) AC#2), and the Liberator's per-page `stardust_sweep_page_{pageId}` lock, held while sweeping one `entry_slots_page_X` table so two workers never nullify the same page at once ([ADR 0049](adrs/0049-multi-worker-liberator-excluded-per-page.md)). Both lock names and their timeouts (10 seconds for provisioning, zero for a sweep) are now normative — not merely code, as this entry previously said. Per [ADR 0053](adrs/0053-advisory-lock-names-are-qualified-per-installation.md), the literal name reaching the server also carries a per-installation suffix derived from the schema name, so unrelated installations sharing one MySQL server never contend on each other's lock — see Lock Namespace.

**See also:** The Watcher, The Liberator, Lock Namespace, Page, [ADR 0008](adrs/0008-singleton-watcher-multi-worker-reconciler.md), [ADR 0049](adrs/0049-multi-worker-liberator-excluded-per-page.md), [ADR 0053](adrs/0053-advisory-lock-names-are-qualified-per-installation.md).

---

### Advisory Schedule

The single-row `stardust_advisory_schedule` table holding the due time of the next Watcher advisory sample (the Coercion Failure-adjacent cardinality and Spread samples), introduced by [ADR 0052](adrs/0052-the-advisory-schedule-is-persisted-and-fleet-wide.md). It breaks the pattern every other `stardust_*` glossary entry follows of one entry per table with a matching structural role, because it is not a coordination contract table like the Schema Registry or a work queue like `stardust_sync_queue` — it exists solely so a process that exits after every invocation (the bounded tick, see Combined Tick) can still find out whether a sample is due, which an in-memory schedule cannot survive. The schedule is fleet-wide: one sample fires per interval across every process sharing the deployment, not one per daemon, decided by a single conditional `UPDATE` whose affected-row count is the claim itself.

**See also:** The Watcher, Spread, Combined Tick, [ADR 0052](adrs/0052-the-advisory-schedule-is-persisted-and-fleet-wide.md), [ADR 0019](adrs/0019-index-cardinality-policy.md).

---

### Anchored Cursor

The pagination-token shape for a sorted read, introduced by [ADR 0041](adrs/0041-sort-ordering-and-the-anchored-cursor.md). It does not carry the sort key's value directly — embedding it was rejected because a string slot's 4096-character value would push a self-contained token past practical URL/header size limits. Instead the token names the anchor row's `entry_id`, the sort key's identity, and the sort direction; the compiler resolves the anchor's actual sort value at query time via a one-row derived table (`CROSS JOIN (SELECT (SELECT <col> ...) AS av) sort_anchor`), which keeps the token constant-size regardless of the sorted field's width. An unsorted read keeps emitting the original `base64url("v1:" . entryId)` token byte-for-byte; a sorted read emits a `base64url("v2:" . json)` token. Both decode, and a v1 token is accepted by an explicitly-default sort, since `null` and ascending-by-id name the same ordering.

**See also:** Cursor-Based Pagination, Sort, [ADR 0041](adrs/0041-sort-ordering-and-the-anchored-cursor.md).

---

### Backfill Pump

**Not yet built** — [ADR 0040](adrs/0040-import-manifest-enumerates-chunks.md) still refers to it as "the (not-yet-built) Backfill Pump CLI." Planned as a CLI command (`bin/stardust backfill`) that would iterate over historical `entry_data` records in ascending `id` order and push them into the event stream for replication into extension tables, for use during legacy data migration. It would maintain state via the same `backfill_checkpoints` table the field-lifecycle work sources already use (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.4), under its own `job_name` namespace with no retype semantics — which is why `backfill_checkpoints.source_declared_type` must stay nullable rather than required. Would allow resumability (`--from-id`) and report throughput metrics to stdout.

**See also:** Dual-Write, Dead Letter Queue, `backfill_checkpoints`, [ADR 0040](adrs/0040-import-manifest-enumerates-chunks.md).

---

### Bounded Fetch

The second query in the Two-Query Approach (Query 2). After the Paginated Probe identifies a bounded set of matching IDs, the Bounded Fetch retrieves the full row payloads via a safe `WHERE id IN (...)` clause with any necessary extension table joins. The input set is always capped at `page_size`, guaranteeing constant memory usage.

**See also:** Paginated Probe, Two-Query Approach.

---

### Bulk Ingestion

The chunked, multi-entry write path specified by [ADR 0011](adrs/0011-chunked-bulk-ingestion.md). A batch is processed in configurable chunks (default 500 entities), each committed in its own transaction — whole-batch atomicity is deliberately not offered, so a mid-batch failure leaves earlier chunks committed and later ones untouched. Two submission modes exist, split by size: **synchronous** (≤ 1,000 entities) runs inline and returns a result enumerating every chunk's outcome directly; **asynchronous** (> 1,000 entities, or an explicit submission) persists a `stardust_import_jobs` row, returns an Import Job ID immediately, and hands the work to The Reconciler's import work source rather than a dedicated daemon. Both modes accept an optional idempotency key so a caller can safely retry across a dropped connection without duplicating entries. Per [ADR 0040](adrs/0040-import-manifest-enumerates-chunks.md), the async job's manifest enumerates each chunk's outcome (`committed | failed`, plus its entity-id range) the same way the synchronous result always has; `rolled_back` is a synchronous-only outcome, since an async chunk failure is terminal for the job rather than skipped and continued past.

**Aliases:** Chunked Bulk Ingestion, Async Bulk Ingest.
**See also:** The Reconciler, `stardust_sync_queue`, Exhaustion Fallback, [ADR 0011](adrs/0011-chunked-bulk-ingestion.md), [ADR 0040](adrs/0040-import-manifest-enumerates-chunks.md).

---

### Core Payload Table

The primary transactional storage table for all entries. Physically named `entry_data`. Stores the complete, unindexed JSON payload in a `fields` column alongside system timestamps and tenant/model identifiers. Every entry in StarDust has exactly one row in this table regardless of extension table state.

**Aliases:** `entry_data`.
**See also:** Extension Table, Payload Splitting Engine.

---

### Cursor-Based Pagination

The pagination strategy enforced by StarDust's function API. Instead of `OFFSET`-based pagination (which degrades at depth), queries use `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. The `+1` row determines whether a next page exists. The system never evaluates the total matched set of a query, ensuring constant-time pagination regardless of dataset size.

**See also:** Paginated Probe, Two-Query Approach, Anchored Cursor, Sort.

---

### Coercion Failure

The Reconciler outcome when a JSON payload value cannot be coerced into the target slot column's type during retype backfill (e.g., `string → int` on the value `"42abc"`, or any of the four categorically-rejected `int↔datetime` / `numeric↔datetime` cells). The slot column receives `NULL`; the JSON payload at `entry_data.fields` remains the authoritative system of record per [ADR 0013](adrs/0013-json-payload-as-system-of-record.md), so reads fall back to `JSON_EXTRACT` without permanent data loss. Coercion failures are explicitly NOT poison pills (they never quarantine to `stardust_reconciler_dlq`); each emits a `coercion_null` structured-log event for operator audit. The full per-type-pair predicate is normative in [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

**See also:** The Reconciler, Tombstoned Slot, [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

---

### Coercion Matrix

The 4×4 lookup table governing how a JSON payload value converts between the four declared types (`string`, `int`, `numeric`, `datetime`) during retype backfill. Each of the twelve off-diagonal cells pins one conversion's predicate and canonical output form, inheriting the QueryFilter typed-value rules (`blueprints/queryfilter_wire_format.md` §4.5) so ingress validation and migration coercion share a single definition of each type. The **identity diagonal** (`string → string`, `int → int`, …) is a no-op pass-through — same-type retypes such as Model Compaction relocations ride it and cannot fail. Four cells are categorically rejected (`int ↔ datetime`, `numeric ↔ datetime`) because epoch interpretation is caller policy, not engine semantics; callers bridge through a `string` intermediate. An attempted-but-failed cell stores `NULL` and emits `coercion_null` (see Coercion Failure). The matrix applies only to the Reconciler retype-backfill path, never to wire-format input validation. Normative in [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md).

**Aliases:** Type Coercion Matrix.
**See also:** Coercion Failure, Model Compaction, The Reconciler, [ADR 0024](adrs/0024-type-coercion-matrix-for-retype-backfill.md), [ADR 0016](adrs/0016-field-type-change-lifecycle.md).

---

### Combined Tick

The bounded, budget-limited daemon run (`bin/stardust tick`) introduced by [ADR 0048](adrs/0048-bounded-combined-tick-for-cron-driven-hosting.md) for a host with no persistent-process capability, such as cron-only shared hosting. One process, one MySQL connection: it takes the Watcher's own singleton lock, runs the Watcher once, then loops sweeping one Liberator batch and one Reconciler round per iteration until a time budget is spent, a round finds nothing to do, or shutdown is requested. It composes three of the four daemons — the Chronicler is opt-in via `--exports`, bounded by its own Cooperative Yield. Because every daemon it composes already checkpoints its state to the database, a bounded pass is structurally equivalent to a daemon that crashes and restarts, which is what makes the run safe to invoke repeatedly from a crontab line or a scheduled URL fetch.

**See also:** The Watcher, The Reconciler, The Liberator, Cooperative Yield, [ADR 0048](adrs/0048-bounded-combined-tick-for-cron-driven-hosting.md).

---

### Cooperative Yield

The mechanism by which an in-progress Chronicler export gives up its claim at a committed chunk boundary rather than running to completion, introduced by [ADR 0050](adrs/0050-chronicler-cooperative-yield-at-a-chunk-boundary.md). After a non-final chunk commits, the worker consults a yield signal (a time budget, in Combined Tick's case, or the daemon's own shutdown signal); if it fires, the same transaction sets the job back to `pending` with `worker_identity = NULL` and emits `job_yielded`, leaving the Resume Anchor intact so any worker can pick the job back up at its next poll. This is what lets an export whose total size is unknown in advance be bounded by a Combined Tick run without either abandoning it mid-file or letting it consume the whole budget.

**See also:** The Chronicler, Combined Tick, Resume Anchor, [ADR 0050](adrs/0050-chronicler-cooperative-yield-at-a-chunk-boundary.md).

---

### Dead Letter Queue (DLQ)

A holding queue for migration event payloads that failed processing by the dual-write consumer worker. Monitored with alerting thresholds: critical alerts fire if DLQ depth exceeds 100 messages or the oldest message age exceeds 12 hours. Failed messages are replayed by re-submitting using the original `entry_id` partition key to preserve causal ordering. Operational detail (the replay tooling itself) is out of scope here — [`legacy_data_migration.md`](legacy_data_migration.md) is a stub pending blueprint stabilization.

**See also:** Dual-Write, Backfill Pump.

---

### Desync Flag

A state indicator signaling that an entry's indexed representation in one or more extension tables is inconsistent with the current schema registry expectations. Two scenarios produce desync, both with documented resolution paths:

1. **Row desync** — The extension table row is entirely missing. This occurs during an Exhaustion Fallback: the entry is written to `entry_data` only, and its `entry_id` is enqueued to `stardust_sync_queue`. The Reconciler resolves this by backfilling the missing row once capacity is restored.
2. **Index desync** — A field's `is_filterable` flag was promoted from `false` to `true`. Under ADR `0034` a non-filterable field is JSON-only and holds no slot, so promotion is normally a fresh indexed-slot reservation plus a backfill from the JSON payload — no eviction. (The legacy variant, where the field already had a populated, unindexed slot from before ADR `0034`, additionally severs and tombstones that grandfathered slot; because `ALTER TABLE` on populated pages is forbidden per [ADR 0012](adrs/0012-immutable-extension-page-ddl.md), the old slot cannot acquire an index in place.) Either way the field undergoes the **filterability-promotion lifecycle** defined in [ADR 0016](adrs/0016-field-type-change-lifecycle.md): a new indexed slot of the field's type is assigned `free → backfilling`, and the Reconciler backfills it from the JSON payload. While the new slot is `backfilling`, reads fall back to `JSON_EXTRACT` and filters are rejected (the field is unmapped per the Schema Registry contract in [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md)). Once promoted to `ready`, indexed filtering resumes.

**See also:** Exhaustion Fallback, The Reconciler, `is_filterable`, Page, [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md).

---

### Driver/Adapter Pattern

The extensibility architecture allowing StarDust's read path to be backed by different search engines without altering the ingestion or API layers. The default implementation is the MySQL Native Driver. Alternative drivers (e.g., a Meilisearch driver) can be injected via the engine's Config object at construction time ([ADR 0026](adrs/0026-framework-neutral-composer-packaging.md)). All drivers implement the `EntrySearchInterface` contract and are strictly read-only — writes always target MySQL.

**See also:** `EntrySearchInterface`, MySQL Native Driver, Consistency Header.

---

### Dual-Write

The asynchronous event-driven replication strategy used during legacy data migration. The API synchronous path writes mutations only to the legacy Virtual Column system, then emits a domain event (`EntryCreated`, `EntryUpdated_v2`, `EntryDeleted`) to a message queue using `entry_id` as the partition key. A background consumer worker processes these events and executes idempotent upserts against the new extension tables. Synchronous atomic dual-writes are strictly forbidden.

**See also:** Virtual Column Method, Dead Letter Queue, Backfill Pump.

---

### `entry_data`

The physical MySQL table name for the Core Payload Table.

**See also:** Core Payload Table.

---

### `EntrySearchInterface`

A PHP interface defining the search driver contract for StarDust's read path. Method signatures cover: filtered listing with cursor pagination, single-entry retrieval, and capability introspection (e.g., `supportsFuzzySearch(): bool`). If a caller requests a capability the active driver does not support, the function API rejects the call with a typed exception.

**See also:** Driver/Adapter Pattern, MySQL Native Driver.

---

### `entry_slots_page_X`

The physical MySQL table name pattern for Extension Tables. `X` is a monotonically increasing page number assigned by the Watcher at provisioning time (e.g., `entry_slots_page_1`, `entry_slots_page_2`).

**See also:** Extension Table, Page.

---

### Entry

A single data record belonging to a model within a tenant. Represented as one row in `entry_data` and optionally one row in each relevant `entry_slots_page_X` table. Identified by `entry_id` (aliased as `id` on `entry_data`).

**See also:** Model, Tenant, Core Payload Table.

---

### Exhaustion Fallback

The graceful degradation behavior triggered when all extension table slot capacity reaches 100% and no pages have available capacity. During exhaustion, the write path: (1) writes the full JSON payload to `entry_data`, (2) skips the extension table write entirely, and (3) enqueues the `entry_id` to `stardust_sync_queue`. This ensures high-throughput ingestion never blocks or drops writes, even when the Watcher has failed to provision new pages.

**See also:** Desync Flag, The Watcher, The Reconciler, `stardust_sync_queue`.

---

### Extension Table

A 1:1 table (physically named `entry_slots_page_X`) that stores explicitly indexed fields extracted from an entry's JSON payload, with a foreign key to `entry_data` (`ON DELETE CASCADE`). New extension tables are dynamically provisioned by the Watcher when slot capacity runs low.

A page is created with **exactly the columns it indexes** — no spare, unclaimable columns — so `free` and `claimable` capacity are the same thing. How many columns that is comes from Index Headroom: at provisioning time each of the four slot families receives `max(demand, k)` indexed columns (`k = Config::$pageIndexHeadroom`, default 4), so a fresh page ordinarily carries sixteen columns. Once created, a page's shape is fixed for life — `ALTER TABLE` on a populated page is forbidden (ADR 0012).

**Aliases:** Extension Page (informal).
**See also:** Page, Slot, Index Headroom, Vertical Schema Partitioning, `entry_slots_page_X`, [ADR 0042](adrs/0042-index-headroom-at-page-provisioning.md), [ADR 0043](adrs/0043-pages-provision-only-indexed-columns.md).

---

### Field Deletion

The definition-layer operation that removes a Field from a Model. **Severance is synchronous and total; the payload purge is asynchronous; the registry row dies last.** `deleteField()` commits one registry transaction and returns: `stardust_fields.deleted_at` is set and `is_filterable` cleared, any live Slot is tombstoned by the two-step sequence that nulls `field_id` before flipping `status` (releasing the `RESTRICT` foreign key inside the transaction), terminal sibling checkpoint rows are removed, the schema version is bumped, and a `running` `delete_field_{id}` checkpoint is opened. From that commit the field is gone from reads, point reads, introspection, filters, new CSV export headers and inbound writes — whose payloads have the key **stripped**, not merely left unmapped, because an unregistered key would otherwise be preserved verbatim and the purge would never converge. The Reconciler then removes the key from `entry_data.fields` in bounded chunks, and the final chunk hard-deletes the `stardust_fields` row, deletes the checkpoint, and bumps the version in one transaction. Under [ADR 0034](adrs/0034-non-filterable-fields-are-json-only.md) the common case holds no slot at all. There is no undelete, and the field's name is not reusable until the purge lands. Specified by [ADR 0037](adrs/0037-field-deletion-lifecycle.md).

**See also:** Model Deletion, Soft Deletion, Tombstoned Slot, Backfill Pump, The Reconciler, [ADR 0037](adrs/0037-field-deletion-lifecycle.md), [ADR 0036](adrs/0036-entry-payload-keys-are-field-names.md).

---

### Field Rename

The definition-layer operation that changes a field's `stardust_fields.name`. Per [ADR 0036](adrs/0036-entry-payload-keys-are-field-names.md), the `entry_data.fields` JSON object is keyed by field **name** — a deliberate decision, not an accident, made because an unregistered key has no `field_id` to key by instead, and because the CSV export header, the JSON export artifact, and the async import artifact are all name-keyed contracts already written to disk by prior deploys. That makes a rename a **payload rewrite**, not a single registry UPDATE the way Model rename is. Initiating a rename sets the new `name` and the old value into `stardust_fields.previous_name` synchronously and returns; the Reconciler then backfills `entry_data.fields` for the model in chunks, riding the same `backfill_checkpoints` table and work-source shape a field retype uses ([ADR 0016](adrs/0016-field-type-change-lifecycle.md)), clearing `previous_name` in the same transaction that completes the drain.

**Reads and writes bridge the rename window; filters do not.** A read falls back to `previous_name` when the new key is absent, and a write canonicalises an old-name key to the new name before persisting, but a filter naming the old field is rejected outright — a rejected write would lose data, while a rejected filter loses nothing, so loud rejection is the correct failure mode only for the filter. A field may have at most one lifecycle in flight: a rename and a retype must not overlap, or the retype backfill would read every un-migrated row as value-absent and silently null the new slot.

**See also:** Field Deletion, Model, Coercion Matrix, [ADR 0036](adrs/0036-entry-payload-keys-are-field-names.md), [ADR 0016](adrs/0016-field-type-change-lifecycle.md).

---

### Feature Flag (Rollback / Dual-Write)

Two decoupled feature flags used during legacy data migration. The **Read Feature Flag** controls which schema serves read traffic (legacy Virtual Columns or new extension tables). The **Dual-Write Feature Flag** controls whether the event producer emits domain events for replication. These flags are intentionally independent: if reads are rolled back to legacy, the event producer must remain active to prevent data divergence in the extension tables.

**See also:** Dual-Write, Virtual Column Method.

---

### `is_filterable`

A boolean metadata flag in the schema registry, set at model-field registration time. When `true`, the field is assigned an extension-table slot whose composite B-tree index `(tenant_id, slot_column)` is included in the page DDL at provisioning time. When `false`, the field is **JSON-only** — it is never assigned a slot (ADR `0034`), lives solely in `entry_data.fields`, is retrieved via `JSON_EXTRACT` on select, and any function-API attempt to filter on it is rejected with a typed exception before the database is touched.

**See also:** Slot, Index Provisioning Policy, Pre-Flight Rejection.

---

### Index Headroom

A provisioning-time setting, `k` (`Config::$pageIndexHeadroom`, default 4), that controls how many extra indexed columns get built into a new page before anything actually needs them.

Here's the problem it solves. Earlier, a new page was given only as many indexed columns as were needed at that exact moment — nothing spare. That sounds efficient, but it backfired: say an operator makes three string fields filterable, one after another. The first field takes the only spare string column on the newest page. The second field, promoted moments later, finds no spare string column anywhere — so the Watcher provisions a **brand new page** just for it. The third field repeats the same thing. Three ordinary fields end up scattered across three separate pages, and a query touching all three now has to join across all three instead of just one (see Spread).

Index Headroom fixes this by always building a small cushion of spare indexed columns — `k` per column type (string, integer, number, date/time) — into every new page, whether or not anything needs them yet. A fresh page ends up with sixteen indexed columns instead of one to three, so the next few fields promoted usually land on that same page rather than forcing a new one. `k` is fixed once a page is created and can't be widened afterward (ADR 0012), so raising the default only helps pages built from that point on.

**See also:** Extension Table, `is_filterable`, Model-Affine Slot Reservation, [ADR 0042](adrs/0042-index-headroom-at-page-provisioning.md), [ADR 0043](adrs/0043-pages-provision-only-indexed-columns.md).

---

### Index Provisioning Policy

The deterministic, schema-driven rules governing which extension table slots receive B-tree indexes. Indexing decisions are tied to the `is_filterable` metadata flag: only slots mapped to fields with `is_filterable = true` are indexed at page creation time. This ensures index provisioning is auditable and never ad-hoc.

**See also:** `is_filterable`, Pre-Flight Rejection, Extension Table, Model-Affine Slot Reservation.

---

### Lock Namespace

The per-installation suffix appended to an Advisory Lock's literal name before it reaches MySQL, introduced by [ADR 0053](adrs/0053-advisory-lock-names-are-qualified-per-installation.md). It exists because `GET_LOCK` names are scoped to the whole MySQL server, not to a schema, so on a shared host running several StarDust installations against one `mysqld`, an unqualified name let one installation's Watcher or Liberator block on a stranger's lock — a liveness defect, worst on the Liberator's per-page lock, since page ids restart at 1 in every installation. The default suffix derives from the connected schema name, so the fix needs no configuration; an operator can override it to keep one lock identity across a database rename, or to deliberately share one across installations.

**See also:** Advisory Lock, The Watcher, The Liberator, [ADR 0053](adrs/0053-advisory-lock-names-are-qualified-per-installation.md).

---

### Model

A user-defined data structure (schema) within a tenant, identified by `model_id`. A model defines the set of fields, their types, and their slot mappings in extension tables. All entries belong to exactly one model.

A model has a lifecycle: it is registered, may be renamed — one committed UPDATE, since identity is `stardust_models.id` and nothing resolves a model by name — and may be deleted (see Model Deletion). Its `id` is globally unique across tenants, and `entry_data.model_id` references it _logically_, with no foreign key: the data plane never foreign-keys into the Schema Registry.

**See also:** Entry, Tenant, Schema Registry, Model Deletion, Model Compaction, Field Rename.

---

### Model Compaction

The operator-initiated operation that cures Spread: it relocates a fragmented Model's live filterable Slots onto a minimal Page set, restoring the few-joins-per-query property the Slot Spread Metric ([ADR 0031](adrs/0031-slot-spread-metric.md)) measures. Mechanically each relocation is a **same-type retype** riding the unmodified field-lifecycle pipeline ([ADR 0016](adrs/0016-field-type-change-lifecycle.md)) through the coercion matrix's identity diagonal — the only new machinery is a registry-only planner that picks target pages, a page-pinned slot reservation (pin-or-fail; compaction never defers), and a CLI (`bin/stardust compact:model`) that orchestrates relocations **sequentially by default**, so at most one field at a time has filters rejected while its new slot backfills (reads fall back to the JSON payload throughout). Crash recovery is re-run: already-relocated fields are no-ops, so the operation converges idempotently. Never scheduled, never automatic — the operator pays the relocation cost per model, exactly where the metric justifies it. Specified by [ADR 0033](adrs/0033-operator-initiated-model-compaction.md).

Per [ADR 0039](adrs/0039-compaction-refuses-to-plan-mid-lifecycle.md), planning itself — `--dry-run` included — refuses with `RetypeInProgressException` while any field of the target model has a running retype checkpoint. A field mid-retype holds one `tombstoned` slot and one `backfilling` slot and is invisible to the planner's population, which is deliberately the same one the ADR 0031 spread sample uses; planning through that gap would report numbers the metric then contradicts. The refusal is transient and self-clearing once the Reconciler drains the checkpoint.

**Aliases:** Compaction.
**See also:** Spread, Model-Affine Slot Reservation, Page, Slot, Tombstoned Slot, [ADR 0033](adrs/0033-operator-initiated-model-compaction.md), [ADR 0031](adrs/0031-slot-spread-metric.md), [ADR 0039](adrs/0039-compaction-refuses-to-plan-mid-lifecycle.md), [ADR 0016](adrs/0016-field-type-change-lifecycle.md), [`runbooks/maintaining_low_spread.md`](runbooks/maintaining_low_spread.md).

---

### Model Deletion

The definition-layer operation that removes a Model, its Fields, and every Entry belonging to it. Structurally the sibling of Field Deletion — **severance now, purge later, registry row last** — but where a field purge rewrites payload _keys_, a model purge destroys _rows_. `deleteModel()` commits one registry transaction and returns: `stardust_models.deleted_at` is set, every field of the model is marked `deleted_at` with `is_filterable` cleared (which makes every existing field-severance guard fire with no new predicates), every live Slot is tombstoned, terminal checkpoint rows are removed for every field, the schema version is bumped once, and a single `running` `delete_model_{id}` checkpoint is opened. The Reconciler then hard-deletes `entry_data` rows for the `(tenant_id, model_id)` partition in bounded chunks — cascading into `entry_slots_page_X` and deleting the matching `stardust_sync_queue` rows in the same transaction — and the final chunk re-asserts severance, deletes the `stardust_models` row (cascading its fields away), deletes the checkpoint, and bumps the version. Writes to a deleting model are **refused**, inverting Field Deletion's strip-don't-reject rule, because there is no residual valid entry to preserve and an accepted write would become a permanent orphan. The purge has **no dead-letter path** by design, and there is no undelete: entries, extension rows, queue rows, field definitions and the model are all gone. Specified by [ADR 0038](adrs/0038-model-deletion-lifecycle.md).

**See also:** Field Deletion, Soft Deletion, Model, Tombstoned Slot, The Reconciler, `stardust_sync_queue`, [ADR 0038](adrs/0038-model-deletion-lifecycle.md), [ADR 0037](adrs/0037-field-deletion-lifecycle.md), [ADR 0018](adrs/0018-reconciler-poison-pill-semantics.md).

---

### Model-Affine Slot Reservation

The slot-reservation policy that biases the slot reserver toward free Slots on Pages that already host a live slot of the **same Model**, falling back to the global-oldest-free order when no affine slot of the required type family exists. It is a bias, not an allocation: a model never reserves or owns whole pages, and an affine page's free slots stay available to every other model and tenant. The policy reduces Spread at its source — keeping a model's filterable slots co-located so filtered reads touch fewer pages — without sacrificing slot density, because it only reorders candidates within the existing free-slot pool and never provisions a page the Watcher would not otherwise create. Affinity is forward-only prevention: it keeps fresh and incrementally-grown models compact, but it does not converge models that are already spread (that is operator-initiated compaction) and cannot beat per-family slot ceilings. Specified by [ADR 0032](adrs/0032-model-affine-slot-reservation.md).

**Aliases:** Slot Affinity.
**See also:** Spread, Model Compaction, Slot, Page, Index Provisioning Policy, [ADR 0032](adrs/0032-model-affine-slot-reservation.md), [ADR 0012](adrs/0012-immutable-extension-page-ddl.md).

---

### MySQL Native Driver

The default implementation of `EntrySearchInterface`. Executes queries directly against MySQL using the strict indexing rules defined in the Index Provisioning Policy. Reports `consistencyModel(): "strong"` (callers translate this into their own consumer-facing surface). Rejects filters on non-indexed fields with a typed exception.

**See also:** `EntrySearchInterface`, Driver/Adapter Pattern, Pre-Flight Rejection.

---

### Page

A single numbered instance of an Extension Table, identified by its numeric suffix (e.g., `entry_slots_page_1`). The term "page" emphasizes the **provisioning lifecycle**: pages are created by the Watcher when global slot capacity drops below the configured threshold, and `ALTER TABLE` on populated pages is strictly forbidden. Slot capacity is tracked at the page level.

**See also:** Extension Table, The Watcher, Slot, Spread.

---

### Paginated Probe

The first query in the Two-Query Approach (Query 1). Executes the filter condition as a standalone query selecting only `id` using covering indexes, bounded by cursor logic: `WHERE id > :cursor ORDER BY id ASC LIMIT {page_size} + 1`. The `+1` determines if a next page exists. This query maintains a constant, tiny memory footprint regardless of total matched rows.

**See also:** Bounded Fetch, Two-Query Approach, Cursor-Based Pagination.

---

### Payload Splitting Engine

The write-path component responsible for separating an entry's data into two destinations: the complete JSON payload goes to `entry_data.fields`, while explicitly indexed fields are extracted and written to the appropriate slot columns in the entry's extension table row.

**See also:** Core Payload Table, Extension Table, Slot.

---

### Pre-Flight Rejection

The function API's strict enforcement mechanism for unindexed filter attempts. If a caller requests a filter or sort on a field lacking `is_filterable = true` in the schema registry, the call is immediately aborted at the API boundary with a typed exception before the database is ever touched. This replaces the earlier "Scanned Row Circuit Breaker" concept.

**See also:** `is_filterable`, Index Provisioning Policy, ~~Scanned Row Circuit Breaker~~.

---

### Resume Anchor

The unit an interrupted Chronicler export resumes from, pinned by [ADR 0047](adrs/0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md) as the artifact file itself plus its verified byte offset — not the `last_cursor` column trusted alone. On an abandoned-claim re-claim or a Cooperative Yield resume, the new worker re-opens the prior partial artifact, verifies it actually holds at least the recorded byte count (and, for CSV, that its header still matches the current field set), and only then truncates to that offset and continues from `last_cursor`. Verification failing — a missing file, a short one, a header mismatch, or an unavailable file lock — discards the anchor and restarts the artifact from byte zero rather than risking a corrupt append.

**See also:** The Chronicler, Cooperative Yield, `stardust_export_jobs`, [ADR 0047](adrs/0047-the-export-resume-anchor-is-the-artifact-plus-its-byte-offset.md).

---

### Scanned Row Circuit Breaker _(deprecated)_

A previously proposed runtime mechanism for bounding query execution by tracking scanned rows during query evaluation. This concept has been superseded by Pre-Flight Rejection — strict schema-level enforcement that categorically prevents unindexed queries at the API contract level, eliminating the need for runtime row-count monitoring.

**Replaced by:** Pre-Flight Rejection. See [ADR 0014](adrs/0014-schema-level-safety-over-runtime-circuit-breaking.md).

---

### Schema Registry

The database-resident metadata catalog that tracks extension table pages, slot assignments, field-to-slot mappings, and `is_filterable` flags. It serves as the sole coordination point between the Watcher, the Reconciler, and the ingestion path — no direct IPC exists between these components. The ingestion path reads the registry (with a configurable cache TTL) to determine which page and slot to write indexed fields to.

**See also:** The Watcher, The Reconciler, `is_filterable`, Page.

---

### Shadow Traffic

An optional pre-cutover validation step during legacy data migration (gate 3 of the Three-Gate Protocol). A small percentage of read traffic (e.g., 5%) is routed to the new extension table schema while the majority continues hitting the legacy system. Operators verify P99 latency parity and zero correctness mismatches before proceeding to full cutover.

**See also:** Three-Gate Protocol, Feature Flag.

---

### Slot

A typed column within an extension table (e.g., `i_str_01`, `i_int_15`, `i_num_03`, `i_dt_07`). Each slot has a fixed data type (`TEXT` for string slots per ADR `0030`, or `BIGINT`, `DOUBLE`, `DATETIME`) and, when occupied, is mapped to a specific **filterable** model field via the schema registry — non-filterable fields are JSON-only and never occupy a slot (ADR `0034`). A page's column set equals its indexed set exactly — there are no spare, unindexed columns sitting on a page — so every column on a page is a slot. The total number of available slots across all pages determines global capacity.

**See also:** Extension Table, Page, `is_filterable`, Schema Registry, Spread, Index Headroom.

---

### Slot Squatting

The capacity exhaustion anti-pattern where fields that are no longer filterable (or entirely deleted) continue to occupy physical column slots in extension tables. The Liberator daemon prevents this by reclaiming and nullifying these slots.

**See also:** The Liberator, Slot, Spread.

---

### Soft Deletion

The temporal deletion strategy used by StarDust for **Entries**. `deleteEntry()` never physically removes a row; instead, the `deleted_at` column on `entry_data` is set to the deletion timestamp. The composite index `(tenant_id, deleted_at, created_at)` supports efficient queries that exclude soft-deleted records.

This is a property of the entry lifecycle, not a guarantee that bytes are never freed. **The one operation that physically deletes `entry_data` rows is the Model Deletion purge** ([ADR 0038](adrs/0038-model-deletion-lifecycle.md)), and it is a definition-layer operation rather than an entry-level one: it destroys the Model those entries were instances of, so there is nothing left for a soft-deleted row to be a soft-deleted instance _of_. Note that the purge carries no `deleted_at IS NULL` predicate — matching the rename and retype drains — so already-soft-deleted entries are removed along with the rest.

Do not confuse this with `stardust_fields.deleted_at` or `stardust_models.deleted_at`. Those are **drain-window markers**, not a soft-delete tier: they mean "a deletion is in flight", there is no undelete, and the purge's final chunk removes the row outright.

**See also:** Entry, Core Payload Table, Field Deletion, Model Deletion.

---

### Sort

The read-path ordering parameter added by [ADR 0041](adrs/0041-sort-ordering-and-the-anchored-cursor.md). `EntryQuery` and `SearchRequest` each carry an appended sort parameter; `null` means `entry_data.id ASC`, the ordering every read had before this ADR, so no existing caller or cursor is affected by its addition. A sort spec names exactly one of three targets: `Id`, `CreatedAt`, or `Field` (a registered field's indexed slot column). Sorting by `Id` or `CreatedAt` can walk straight down an existing index in the requested order — cheap, at any page depth. Sorting by `Field` cannot, because the compiled query reads from `entry_data` first, so MySQL falls back to a **filesort**: instead of reading rows already in order off an index, it gathers all the matching rows first and then sorts that whole set as a separate step. (This is MySQL's own term — it shows up as `Using filesort` in `EXPLAIN` output, and despite the name it doesn't necessarily touch disk; for a small result it sorts in memory.) A field sort still respects the two-query bound — nothing unbounded is ever materialized — but it costs a full sort pass over the filtered set on every page, rather than a cheap index walk. `entry_data.id` is always appended as the implicit tiebreak in the same direction as the requested sort, which is what keeps every ordering total and every Anchored Cursor stable. Sortability on a given field is a driver capability — `EntrySearchInterface::supportsSortOn(int $fieldId): bool` mirrors `supportsFilterOn()` but is a separate method, since an external driver may index a field for matching without making it orderable. Sort does **not** enter the QueryFilter wire format; it is a parameter on the read DTOs only, per the 2026-05-02 confirmation that sort belongs to the consumer API layer.

**Aliases:** SortSpec.
**See also:** Anchored Cursor, Cursor-Based Pagination, Pre-Flight Rejection, `EntrySearchInterface`, [ADR 0041](adrs/0041-sort-ordering-and-the-anchored-cursor.md).

---

### Spread

How many separate extension Pages a single Model's live filterable Slots are scattered across. This matters because every extra page a filtered query touches costs one more join — a model whose fields sit on three pages is slower per query than the same model packed onto one page. Spread isn't a bug; it's a side effect of two things working as designed: a page can never be altered once created (see Page), and a new slot is always handed out from whichever page happens to have room first. A model's fields can end up scattered simply through the ordinary process of growing over time.

StarDust measures this with the advisory **Slot Spread Metric** (ADR 0031): for each model, **excess pages** = pages the model actually occupies − the fewest pages it could theoretically fit on. That "fewest possible" number used to be a simple division, back when every page had the exact same number of columns of each type. It no longer is one, now that pages can be built with different amounts of room (see Index Headroom) — the fewest-pages number has to be worked out by looking at the model's actual pages and how much real spare room each one has, not by a fixed formula (ADR 0044). One side effect: this number can shift on its own, with nothing about the model itself changing, if some other model sharing the same page claims or frees up room there.

The metric never blocks or rewrites anything on its own — it is purely advisory, and fixing a spread-out model is something an operator chooses to do (see Model Compaction). Model-Affine Slot Reservation is the preventive half: it tries to keep a model's new slots on pages it already occupies, so spread doesn't happen in the first place.

**Aliases:** Slot Spread, Excess Pages.
**See also:** Page, Slot, Model, Model-Affine Slot Reservation, Model Compaction, Index Headroom, The Watcher, [ADR 0031](adrs/0031-slot-spread-metric.md), [ADR 0012](adrs/0012-immutable-extension-page-ddl.md), [ADR 0044](adrs/0044-theoretical-minimum-pages-from-real-capacity.md).

---

### `stardust_export_jobs`

The physical MySQL table name for the Chronicler's async export job queue (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.2). Rows transition through `pending → processing → completed | failed`; Chronicler workers claim pending rows via `SELECT ... FOR UPDATE SKIP LOCKED` ordered by per-tenant round-robin position. The 24-hour artifact TTL, 5 GB artifact cap, and ≤ 3 active jobs per tenant are policy from [ADR 0010](adrs/0010-asynchronous-exports.md); daemon-side claim and GC semantics live in [`blueprints/chronicler_daemon.md`](blueprints/chronicler_daemon.md).

**See also:** The Chronicler, Cursor-Based Pagination, Two-Query Approach.

---

### `stardust_reconciler_dlq`

The physical MySQL table name for the Reconciler's per-row poison-pill quarantine (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.3). One row per quarantined entry, distinguished by a `source` discriminator (`sync_queue` or `bulk_import`) so the two Reconciler workloads share one operator surface. Replay is operator-initiated only, via a dedicated CLI replay command; there is no automatic retry and no automatic TTL. Fully specified by [ADR 0018](adrs/0018-reconciler-poison-pill-semantics.md). Distinct from the migration **Dead Letter Queue (DLQ)** above, which holds dual-write replication failures rather than indexed-materialization failures.

**See also:** The Reconciler, Dead Letter Queue, [ADR 0018](adrs/0018-reconciler-poison-pill-semantics.md).

---

### `stardust_schema_version`

The physical MySQL table name for the registry version counter (see [`schemas/schema_reference.md`](schemas/schema_reference.md) §5.1). A single-row singleton holding a monotonically increasing `version` column. Every transaction that mutates coordination-relevant registry state — page provisioning, slot-status transitions, field metadata changes — increments `version` in the same transaction as the underlying mutation. Read on every API write path as the cache-invalidation token defined by [ADR 0015](adrs/0015-database-as-sole-daemon-coordination-point.md).

**See also:** Schema Registry, [ADR 0015](adrs/0015-database-as-sole-daemon-coordination-point.md), [ADR 0017](adrs/0017-schema-registry-as-coordination-contract.md).

---

### `stardust_sync_queue`

The physical MySQL table name for the ephemeral operations queue. A tiny, dedicated table exclusively for queuing writes that failed due to extension capacity exhaustion. Rows are claimed by the Reconciler via `SELECT ... FOR UPDATE SKIP LOCKED` and deleted after successful backfill. The presence of an `entry_id` in this table is the implicit signal for row-level desync.

**See also:** Exhaustion Fallback, The Reconciler, Desync Flag.

---

### Strict Projection Rule

The architectural directive stating that extension tables are treated strictly as temporal index materializations designed for fast retrieval. The true and authoritative "system of record" always remains the full JSON payload stored within `entry_data.fields`.

**See also:** Extension Table, Core Payload Table.

---

### Tenant

The top-level data isolation boundary in StarDust, identified by `tenant_id`. All queries enforce `tenant_id` matching across `entry_data` and extension table `INNER JOIN` conditions. A tenant's data is completely invisible to other tenants at the query level.

**See also:** Entry, Model.

---

### The Chronicler

An independent background PHP daemon responsible exclusively for materializing export jobs. It claims pending jobs from the exports queue via `SELECT ... FOR UPDATE SKIP LOCKED`, pages through the database using the same Cursor-Based Pagination and Two-Query Approach used by the synchronous read path, and streams the output to a local file on disk (CSV or JSON). Each database operation remains bounded by `page_size`, ensuring the transactional database is never subjected to unbounded queries during export materialization. The Chronicler is independent of the Watcher, Reconciler, and Liberator — it does not participate in daemon-to-daemon coordination (ADR `0015`). It reads `stardust_fields` once per claimed job to derive the CSV header column list (a field-name catalog lookup, not coordination); beyond that one read it never touches the schema registry's slot/page state. How export jobs are _submitted_ (HTTP endpoint, request shape, polling) is the caller's domain.

**See also:** Cursor-Based Pagination, Two-Query Approach, [`blueprints/chronicler_daemon.md`](blueprints/chronicler_daemon.md).

---

### The Liberator

A background PHP CLI daemon (`bin/stardust liberator`) responsible exclusively for sweeping dead or demoted slots to prevent Slot Squatting. It monitors the schema registry for tombstoned slots and reclaims them using chunked DML nullification, ultimately marking them as safely `free` for future indexing needs. Since [ADR 0049](adrs/0049-multi-worker-liberator-excluded-per-page.md), multiple Liberator processes can run at once — it is neither a strict singleton like the Watcher nor unconstrained multi-worker like the Reconciler, but a third category: any number of workers may run, excluded from each other only at page-table granularity via a per-page Advisory Lock, so two workers never sweep the same `entry_slots_page_X` table concurrently.

**See also:** Slot Squatting, Tombstoned Slot, Schema Registry, Advisory Lock.

---

### The Reconciler

An independent, multi-worker background PHP CLI daemon (`bin/stardust reconciler`) responsible for draining the `stardust_sync_queue` and backfilling entries into extension tables. It claims queue rows via `SELECT ... FOR UPDATE SKIP LOCKED` (enabling horizontal scaling), reads the authoritative `entry_data.fields` payload at upsert time (never a stale snapshot), and writes using `INSERT ... ON DUPLICATE KEY UPDATE`. Processes in configurable chunks with inter-chunk delay to prevent write spikes during recovery.

**See also:** The Watcher, Exhaustion Fallback, `stardust_sync_queue`.

---

### The Watcher

A singleton background PHP CLI daemon (`bin/stardust watcher`) responsible for monitoring global slot consumption across all extension tables and provisioning new pages when available capacity drops below the configured threshold (default: 20%). It employs advisory locking, empty-table-only DDL, and atomic registry updates to prevent metadata lock contention. The Watcher's registry update is the sole signal consumed by the Reconciler — no direct notification channel exists.

**See also:** The Reconciler, Page, Advisory Lock, Schema Registry, Spread.

---

### Three-Gate Protocol

The sequential cutover validation process during legacy data migration. Cutover proceeds through three quantifiable gates: (1) **Stream Drain** — consumer group lag holds at `0` for ≥15 minutes; (2) **Data Parity** — random-sample dual-read of ≥10,000 entries yields 100% byte-identical match; (3) **Shadow Traffic** _(optional)_ — a small percentage of reads are routed to the new schema to verify latency and correctness.

**See also:** Dual-Write, Shadow Traffic, Feature Flag.

---

### Tombstoned Slot

The transitional state of an evicted slot. When a field loses its filterable status, its associated slot is severed in the schema registry and marked as "tombstoned". To prevent data bleeding, it cannot be mapped to a new field until The Liberator successfully processes and nullifies all residual data across the respective tenant partition.

**See also:** The Liberator, Slot Squatting, Schema Registry.

---

### Two-Query Approach

The bounded query execution strategy that prevents disk spillage and uncontrolled memory growth on cross-page queries. Consists of two steps: (1) the Paginated Probe (Query 1) selects only `id` values using covering indexes with cursor-based bounds; (2) the Bounded Fetch (Query 2) retrieves full row payloads for the bounded ID set via `WHERE id IN (...)`. This ensures InnoDB never materializes unbounded intermediate result sets.

**Aliases:** Deterministic Late Row Lookups.
**See also:** Paginated Probe, Bounded Fetch, Cursor-Based Pagination.

---

### Vertical Schema Partitioning

The overarching architectural strategy of StarDust. Instead of storing all data in a single wide table (the legacy Virtual Column Method), entries are split across a Core Payload Table (`entry_data`) for the complete JSON payload and 1:1 Extension Tables (`entry_slots_page_X`) for explicitly indexed fields. This separation allows independent scaling of storage and indexing concerns.

**See also:** Extension Table, Core Payload Table, Virtual Column Method.

---

### Virtual Column Method

The legacy single-table architecture being migrated away from. In this approach, all indexed fields were represented as generated virtual columns on a single table, leading to schema rigidity and performance degradation at scale. StarDust's migration to Vertical Schema Partitioning replaces this approach.

**See also:** Vertical Schema Partitioning, Dual-Write, Legacy Data Migration.
