# 0004 - Fail Fast on Unindexed Filters

**Status:** Proposed
**Created:** 2026-04-14

## Context

If StarDust silently permits filters on non-indexed fields, the database may degrade into table scans, `JSON_EXTRACT` predicates, temporary table materialization, or other expensive behaviors that are hard to predict and easy to normalize. That kind of permissiveness would make API behavior appear flexible while pushing risk into database latency, tenant isolation, and operational stability. The project needs an explicit contract for what happens when consumers ask for queries the native engine was not designed to support.

## Decision

The default MySQL-native API will reject filters and sorts on fields that are not explicitly provisioned as queryable. Requests that violate this rule will fail before touching the database and return `400 Bad Request`.

We will not provide transparent fallback behavior such as `JSON_EXTRACT` filtering, best-effort scans, in-memory post-filtering, or silent routing to a different consistency model. If a field must participate in filtering, that requirement must first be reflected in the schema and index provisioning policy.

## Consequences

**Positive:**

- Database safety is preserved by preventing unsupported query shapes from ever reaching MySQL.
- API behavior is explicit and predictable: supported queries work within bounds, unsupported ones fail clearly.
- The rule reinforces healthy schema design by forcing teams to decide which filters are real business requirements.
- Operators avoid the false confidence that comes from "working" queries whose performance collapses at scale.

**Negative:**

- Consumers may perceive the API as less flexible, especially during early schema evolution.
- Product teams must do additional upfront design work when introducing new filter requirements.
- Some one-off reporting use cases must use alternate paths, such as asynchronous exports or future external search drivers, rather than the synchronous API.
