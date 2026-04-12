# SDDPG — StarDust Developer & Project Guide

> **This repository contains developer-facing documentation for the StarDust project.**
> It is **not** end-user documentation.

## Purpose

SDDPG serves as the canonical reference for architectural decisions, migration strategies, operational procedures, and domain knowledge related to StarDust. Its audience is:

- **StarDust core developers** — day-to-day technical reference.
- **Internal team members** — onboarding context and project history.
- **Future contributors** — architectural rationale and operational playbooks.

## Current Contents

| Document                                                     | Description                                                                                                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`architecture_blueprint.md`](architecture_blueprint.md)     | Core Architecture Blueprint — covers Vertical Schema Partitioning, strict resource bounding, API contracts, and read/write paths.                |
| [`schemas/schema_reference.md`](schemas/schema_reference.md) | ERD & Schema Reference — the single source of truth for the physical schema (core payload, extension tables, and operations queue).              |
| [`legacy_data_migration.md`](legacy_data_migration.md)       | Operational playbook for migrating historical data — covers asynchronous dual-writes, dead letter queues, backfill pumps, and cutover protocols. |
| [`blueprints/`](blueprints/)                                 | Feature Blueprints — high-level feature descriptions and acceptance criteria, written before implementation begins.                              |
| [`glossary.md`](glossary.md)                                 | Domain Dictionary — canonical definitions for project-specific terms (e.g., "extension table", "slot", "page"). Eliminates cross-team ambiguity. |
| [`adrs/`](adrs/)                                             | Architecture Decision Records — immutable log of _why_ key technical decisions were made. Prevents re-litigating settled debates.                |

## Planned Contents

| Document / Directory                    | Purpose                                                                                                                                                  |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api/` — API Contract Specification     | Internal API surface documentation — endpoint behavior, error codes, consistency headers. A developer's behavioral reference, not a public OpenAPI spec. |
| `runbooks/` — Ops Playbook              | Operational procedures: DLQ replay, backfill pump execution, page provisioning, rollback triggers. Reduces bus factor.                                   |
| `onboarding.md` — Onboarding Guide      | Step-by-step guide for a new developer to set up, understand, and contribute to StarDust.                                                                |

## Repository Structure

```text
SDDPG/
├── README.md
├── architecture_blueprint.md
├── legacy_data_migration.md
├── adrs/               # Architecture Decision Records
├── schemas/            # ERD, schema diagrams
├── api/                # API contract docs
├── runbooks/           # Operational playbooks
├── blueprints/         # Feature blueprints & specs
├── glossary.md         # Domain dictionary
└── onboarding.md       # New developer guide
```

## Conventions

- **File format**: Markdown (`.md`). Use Mermaid fenced blocks for diagrams where possible.
- **Naming**: lowercase with underscores (e.g., `migration_plan.md`, not `MigrationPlan.md`).
- **ADR numbering**: `NNNN-short-title.md` (e.g., `0001-extension-tables-over-eav.md`).
- **Immutability**: ADRs are append-only. To supersede a decision, create a new ADR referencing the old one — never edit the original.
