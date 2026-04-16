# 0008 - Singleton Watcher with Multi-Worker Reconciler

**Status:** Accepted
**Created:** 2026-04-17

## Context

StarDust's extension-table lifecycle is managed by two independent background daemons: the Watcher (capacity provisioner) and the Reconciler (queue drain and backfill). Both are long-running PHP processes, but they operate on fundamentally different resources with different contention profiles.

The Watcher provisions new `entry_slots_page_X` tables via DDL when global slot capacity drops below its threshold. This operation involves `CREATE TABLE`, index creation, advisory locking (`GET_LOCK`), and atomic registry updates. DDL execution acquires metadata locks that affect the entire table namespace, and advisory locks in MySQL are connection-scoped singletons — two concurrent Watchers racing to provision the same page will encounter lock timeouts, duplicate table names, or partially committed registry state even when both correctly attempt `GET_LOCK` before proceeding.

The Reconciler drains the `stardust_sync_queue` by claiming rows via `SELECT ... FOR UPDATE SKIP LOCKED` and backfilling extension table slots. `SKIP LOCKED` provides natural row-level mutual exclusion: each worker atomically claims a disjoint subset of queue rows without blocking other workers or the ingestion path. No two workers ever process the same row, and no coordination beyond the database itself is required.

Operators encountering a large backfill queue will naturally want to scale the Reconciler horizontally. Without documenting the asymmetry, the same operators may attempt to scale the Watcher — introducing DDL races that are difficult to diagnose because they manifest as intermittent lock timeouts or silent duplicate-table errors rather than immediate crashes.

## Decision

The Watcher is deployed as a strict singleton process. Only one Watcher instance may run at any time, enforced via PID file or OS-level process locking. No application-level leader election or external coordination service is introduced — this would violate the zero-dependency core (ADR `0002`).

The Reconciler supports horizontal scaling. Multiple Reconciler workers may run concurrently; `SELECT ... FOR UPDATE SKIP LOCKED` guarantees that each worker claims a disjoint set of queue rows. Workers process in configurable chunks with a configurable delay between chunks to prevent write spikes against extension tables during recovery bursts.

The two daemons share no direct IPC. The schema registry (database) is the sole coordination point: the Watcher's atomic registry update is the signal the Reconciler consumes to discover newly available capacity.

## Consequences

**Positive:**

- The Watcher's singleton constraint eliminates DDL race conditions, duplicate table names, and advisory lock contention — failure modes that are intermittent and extremely difficult to reproduce or debug.
- The Reconciler scales linearly with worker count during recovery bursts, limited only by database write throughput and queue depth. Operators can add workers to drain a large backlog without coordination overhead.
- No external coordinator (ZooKeeper, Redis, etcd) is required for either daemon, preserving the zero-dependency core.
- Failure isolation is preserved: either daemon can crash independently without affecting the other or blocking ingestion.

**Negative:**

- The Watcher is a single point of throughput for provisioning. If provisioning demand exceeds what one Watcher cycle can deliver, the system must wait for successive cycles rather than parallelizing DDL.
- Operators must enforce the singleton constraint operationally (PID file, systemd, container orchestrator). The system does not self-heal if two Watcher instances are accidentally started.
- The asymmetry is non-obvious. Without this documentation, the different scaling profiles of two superficially similar daemons would be a recurring source of operational confusion.

**Rejected alternatives:**

- Multiple competing Watchers with advisory locking alone — advisory locks prevent concurrent execution of the critical section but do not prevent race conditions in the detection phase. Two Watchers can independently conclude that a new page is needed, both attempt `GET_LOCK`, one wins and provisions, and the second either times out or provisions a duplicate.
- Leader election via external coordinator (ZooKeeper, etcd) — introduces an infrastructure dependency and a new failure domain for a process that runs on a leisurely 60-second polling interval. The operational cost of the coordinator exceeds the cost of enforcing a PID file.
- Single-worker Reconciler — creates an artificial bottleneck during recovery bursts. A prolonged Watcher outage can accumulate thousands of queue entries; draining them serially delays restoration of indexed query completeness unnecessarily.
- Application-level partitioning of the queue — unnecessary complexity. `SKIP LOCKED` already provides row-level mutual exclusion natively in MySQL without requiring the application to maintain partition assignments or rebalance on worker failure.
