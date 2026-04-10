# Feature Blueprints

> **A blueprint is a high-level feature description and acceptance criteria, written _before_ implementation begins.**

## When to Write a Blueprint

Write a blueprint when a proposed change:

- Spans multiple components or services.
- Introduces or modifies an API contract.
- Requires architectural trade-off decisions that should be reviewed before code is written.

Small, isolated bug fixes and refactors do **not** need a blueprint.

## Conventions

| Rule             | Convention                                                          |
| :--------------- | :------------------------------------------------------------------ |
| File format      | Markdown (`.md`)                                                    |
| File naming      | `<short_snake_case_title>.md`                                       |
| Template         | [`_template.md`](_template.md) — copy this to start a new blueprint |
| Diagrams         | Mermaid fenced blocks                                               |
| Status lifecycle | `Draft` → `Accepted` → `Implemented` → `Superseded`                 |

Blueprints may be updated freely until status reaches `Implemented`. After that, create a new blueprint and mark the old one `Superseded`.

## Index

| Blueprint                                                        | Status | Summary                                                          |
| :--------------------------------------------------------------- | :----- | :--------------------------------------------------------------- |
| [`watcher_reconciler_daemons.md`](watcher_reconciler_daemons.md) | Draft  | Feature spec for the Watcher and Reconciler background daemons.  |
| [`search_driver_adapter.md`](search_driver_adapter.md)           | Draft  | Pluggable search driver interface for non-MySQL search backends. |
