# Legacy Data Migration: Asynchronous Atomicity

This document outlines the operational playbook for migrating existing tenants to the new Vertical Schema Partitioning architecture without locking or threatening the availability of the legacy Virtual Column system.

## 0. Migration Context & Paradigm Shifts

This migration represents a fundamental shift in how the database is utilized to prevent runaway resource consumption at scale. The key transitional shifts are:

- **The Architectural Shift**: This operational playbook governs the active migration from the legacy **Virtual Column Method** (single-table logic) to a 1:1 **Vertical Schema Partitioning (Extension Tables)** strategy (`entry_data` + `entry_slots_page_X`).
- **Mitigating Strict Page Limits**: To enforce database safety without punishing valid consumer use cases, this migration **introduces a dedicated asynchronous export pattern** (`/api/exports`). This allows us to mandate strict, synchronous cursor-based page limits on our core REST API (`/api/entries`) without abandoning consumers who genuinely rely on bulk data retrieval during the transition.

## 1. Asynchronous Dual-Write (Event Stream)

Synchronous atomic dual-writes are strictly forbidden.

- **Phase 1 (The Producer)**: The API synchronous path writes mutations _only_ to the legacy Virtual Columns. Upon success, it emits a domain event (`EntryCreated`, `EntryUpdated_v2`, `EntryDeleted`) to a highly available message queue using `entry_id` as the partition key.
- **Phase 2 (The Consumer)**: A strictly ordered background worker consumes events and executes idempotent upserts against the new `entry_slots_page_X` tables.

## 2. Dead Letter Queue (DLQ) Lifecycle & Observability

If a mapping error occurs, the worker pushes the payload to a DLQ.

- **Alerting**: The system must trigger critical alerts if the DLQ depth exceeds **100 messages** or if the oldest message age exceeds **12 hours**.
- **Replay Mechanism**: Operators use a dedicated CLI command (`php spark stardust:dlq:replay`) to systematically replay failed messages back into the event stream. The replay worker MUST re-submit messages using the exact same `entry_id` partition key to maintain causal ordering against newly arriving updates.

## 3. Idempotent & Resumable Backfill Pump

A background CLI command iterates over historical records, querying `entry_data` in ascending `id` order, and pushing them into the Event Stream to backfill the extension tables. It maintains state via a `backfill_checkpoints` table for resumability (`--from-id`) and reports throughput metrics to stdout.

- **Freshness Guarantee**: The consumer worker must always read the authoritative `entry_data.fields` at upsert time, not the event payload snapshot. This prevents stale backfill events from overwriting fresher real-time updates for the same `entry_id`.

## 4. Validation & Cutover (Three-Gate Protocol)

Cutover proceeds sequentially through three quantifiable gates:

1. **Stream Drain**: Consumer group lag is `0` continuously for ≥ **15 minutes**.
2. **Data Parity**: Random-sample dual-read of ≥ 10,000 entries (legacy vs. extension) yields a 100% byte-identical match after key-sorting.
3. **Shadow Traffic** _(optional)_: Route 5% of reads to the new schema; verify P99 latency and 0 correctness mismatches.

## 5. Safe Rollback Protocol

If anomalies are detected post-cutover, an instant rollback to legacy Virtual Columns is executed via a feature flag.

- **Decoupled Operation**: The Rollback (Read) Feature Flag is completely decoupled from the Dual-Write (Producer) Feature Flag.
- **Shadow Consistency Guarantee**: If read traffic is forcefully reverted to the legacy system, the Event Producer **MUST REMAIN ACTIVE** for all incoming legacy writes. This ensures the experimental extension tables continue to asynchronously shadow the legacy state, preventing data divergence and enabling a future re-cutover attempt without requiring a massive, multi-hour database backfill.
