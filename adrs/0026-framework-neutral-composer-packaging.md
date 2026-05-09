# 0026 - Framework-Neutral Composer Packaging

**Status:** Proposed
**Created:** 2026-05-09

## Context

StarDust ships as a Composer library consumed by callers such as StarGate. Earlier blueprints, the Architecture Blueprint, the glossary, and prior ADRs referenced CodeIgniter 4 specifics — `php spark` command syntax, "CodeIgniter 4 query builder," "CI4 service binding" for search-driver resolution. These references implicitly coupled the engine to a single PHP framework.

The question raised during the implementation-readiness review: should StarDust depend on CodeIgniter 4 directly, or ship framework-neutral with opt-in framework adapters? A direct CI4 dependency would shorten the path to a working v0 by inheriting CI4's query builder, CLI runner, migration tooling, and service container. A framework-neutral packaging would honor the literal reading of [ADR 0002](0002-mysql-native-zero-dependency-core.md) ("zero-dependency core") and keep the engine usable by any framework — or none.

## Decision

StarDust ships as a **framework-neutral Composer library**.

- The engine `composer.json` depends only on PHP 8.x, `ext-pdo`, `ext-pdo_mysql`, `psr/log`, and `psr/clock`. No framework runtime is a hard dependency.
- CLI commands run via a single `bin/stardust` entry point. The underlying console library (e.g., `symfony/console`) is implementation detail and not part of this decision; what is normative is the binary's path and command surface (`bin/stardust watcher`, `bin/stardust reconciler`, `bin/stardust liberator`, `bin/stardust chronicler`, plus operator commands such as `bin/stardust reconciler:dlq:replay` and `bin/stardust backfill`).
- Command syntax in earlier blueprints, the glossary, the schema reference, and any prior ADR (notably [ADR 0018](0018-reconciler-poison-pill-semantics.md)) that uses `php spark stardust:<command>` is superseded by `bin/stardust <command>`. Earlier ADRs are not amended in place per the append-only convention; this ADR is the supersession marker for the CLI-syntax convention.
- Framework-specific service providers — e.g., a CodeIgniter 4 adapter that registers `bin/stardust` commands as `spark` commands and binds the engine into `Config\Services` — are opt-in. They live as adapters within the StarDust monorepo or as separate companion packages, never as core dependencies. The first such adapter is expected to be the CI4 one consumed by StarGate.
- Driver resolution for the search-driver adapter pattern (see [`blueprints/search_driver_adapter.md`](../blueprints/search_driver_adapter.md)) is performed via the engine's public construction API — concretely, a Config-object field accepted at engine instantiation — not via a framework-specific service container. The exact field shape is implementation detail; the architectural commitment is that no framework container is required to select a driver.

## Consequences

**Easier:**

- Any PHP application — vanilla, Laravel, Symfony, CodeIgniter, plain scripts — can consume StarDust without forking.
- Conformance with [ADR 0002](0002-mysql-native-zero-dependency-core.md)'s "zero-dependency core" reading becomes literal, not aspirational.
- Tests construct the engine directly via its public API without bootstrapping a framework or container.
- Each framework gets idiomatic integration through a small adapter, paid for only by consumers who want it.

**Harder:**

- The engine must ship its own thin query layer (for the bounded read path, schema-registry queries, idempotent upserts), CLI bootstrap, and migration runner — each previously available "for free" had CI4 been an accepted dependency. Implementation detail of these layers is left to the implementer; the architectural commitment is that none of them introduce new framework coupling.
- Framework integrations (CI4 first; others if consumers materialize) require small adapter packages or sub-packages. Each adapter is small but adds maintenance surface.

**Trade-offs:**

- Time-to-first-working-version is slightly longer in exchange for no future "extract framework dependency" rewrite.
- Consumers who want CI4 ergonomics must explicitly register the adapter; the cost of one bootstrap line is preferred over hardcoding the framework into the engine.
- This decision pins framework neutrality but deliberately does not pin the full Config-object shape, exception hierarchy, or namespace root — those remain implementation concerns outside SDDPG's scope.
