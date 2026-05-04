# 0002 - MySQL-Native Zero-Dependency Core

**Status:** Proposed
**Created:** 2026-04-12

## Context

StarDust is intended to be deployable as a standalone system with a low infrastructure barrier to entry. Requiring a dedicated search stack such as OpenSearch or Meilisearch from the outset would increase operational complexity, widen the failure surface, and force every deployment into eventual-consistency trade-offs even when those capabilities are not needed. At the same time, the system must leave room for more advanced search integrations in the future.

## Decision

We will ship the core StarDust architecture as a MySQL-native engine with zero mandatory third-party infrastructure dependencies. MySQL is treated as the authoritative transactional store for ingestion and the default bounded read engine for synchronous read access.

Advanced search engines remain optional, not foundational. If a deployment later needs capabilities beyond the native read path, those concerns will be introduced through a separate driver or adapter boundary rather than by changing the core write model or making external search infrastructure a prerequisite for the base product.

## Consequences

**Positive:**

- Standalone deployments remain simple to operate, easier to adopt, and cheaper to run.
- The default consistency model stays straightforward because reads and writes share the same authoritative store.
- The core architecture can be validated and evolved without coupling every feature to an external indexing pipeline.
- Optional integrations remain possible later without redefining StarDust's baseline operational model.

**Negative:**

- The native engine deliberately gives up capabilities that specialized search systems provide, such as fuzzy search or rich full-text features.
- Architectural discipline is required to keep MySQL within a bounded role rather than letting the system drift into unstructured reporting behavior.
- Teams with enterprise search needs may need additional adapter work instead of receiving those features out of the box.
