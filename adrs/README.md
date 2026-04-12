# Architecture Decision Records (ADRs)

> **An immutable log of _why_ key technical decisions were made.**

## Purpose

This directory contains Architecture Decision Records (ADRs) for the StarDust project. ADRs document the context, decisions, and consequences for significant architectural choices (e.g., "Why extension tables over EAV?"). They serve as institutional memory, preventing the team from re-litigating settled debates and providing historical context for future maintainers.

## When to Write an ADR

Write an ADR when a proposed change or decision:

- Introduces new architectural patterns or paradigms.
- Has a massive impact on performance, operability, scalability, or maintainability.
- Replaces or significantly modifies an existing core architectural decision.
- Requires a deliberate trade-off between competing technical goals.

## Conventions

| Rule             | Convention                                                                                                                                                                                                                  |
| :--------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File format**  | Markdown (`.md`)                                                                                                                                                                                                            |
| **File naming**  | `NNNN-short-title.md` (e.g., `0001-extension-tables-over-eav.md`)                                                                                                                                                           |
| **Template**     | [`_template.md`](_template.md) — copy this to start a new ADR                                                                                                                                                               |
| **Immutability** | ADRs are **append-only**. Do not edit the core decision or context of a published ADR. If a decision is changed later, create a new ADR that supersedes the original, and update the status of the old one to `Superseded`. |
| **Status**       | `Proposed` → `Accepted`, `Rejected`, `Deprecated`, `Superseded`                                                                                                                                                             |
