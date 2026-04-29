# 0021 - Search Driver Query Representation: Native QueryFilter Object

**Status:** Proposed
**Created:** 2026-04-23

## Context

The `EntrySearchInterface` is StarDust's extensibility seam for non-MySQL search backends (e.g., Meilisearch, OpenSearch). The blueprint for the search driver adapter ([`search_driver_adapter.md`](../blueprints/search_driver_adapter.md)) defined the surface but left one fundamental question open: **how does a consumer's filter request reach the active driver?**

Two designs are in tension:

- **Native `QueryFilter` value object.** The API parses the consumer's filter into a structured StarDust-internal object (e.g., `QueryFilter(field: "status", op: "in", values: ["open", "pending"])`) and hands that object to the driver. Drivers translate it to their backend's native query language. The current `MysqlNativeDriver` already operates this way internally — the public seam just makes it explicit.
- **Raw DSL string.** The API passes the consumer's raw query string through to the driver verbatim. The MySQL driver parses StarDust DSL; a Meilisearch driver parses Meilisearch query syntax; etc.

The DSL-passthrough route is superficially appealing because it lets each driver expose its full backend capability. But it inverts the architecture: the consumer must know which driver is active to know which DSL to write, and the API loses any structural understanding of the request — breaking the pre-flight rejection layer (ADR `0004`), the schema-validation cache, and the structured-log event vocabulary that all need to inspect filter shape.

## Decision

`EntrySearchInterface` accepts a StarDust-native `QueryFilter` value object. Drivers translate it into their backend's native form. Raw DSL passthrough is rejected.

### `QueryFilter` Shape

The object is a tree of typed nodes. The closed leaf-operator set for v1:

| Operator                 | Semantics                                                                   |
| :----------------------- | :-------------------------------------------------------------------------- |
| `eq`, `neq`              | Equality / inequality on a single value.                                    |
| `lt`, `lte`, `gt`, `gte` | Numeric/datetime comparisons.                                               |
| `in`, `nin`              | Set membership / non-membership.                                            |
| `prefix`                 | String prefix match. Drivers MAY decline via the driver capability surface. |
| `between`                | Inclusive range on numeric/datetime.                                        |
| `is_null`, `is_not_null` | Presence check on the JSON payload.                                         |

Composite nodes: `and`, `or`, `not`. Drivers MUST support all composite nodes — there is no opt-out for boolean composition.

Each leaf node carries the field's `(model_id, field_name)` reference, the operator, and a typed value. The API resolves field references against the schema registry before invoking the driver, so a driver never sees an unknown field. Drivers that support backend-specific capabilities (fuzzy match, faceting, geospatial) declare them through a driver capability surface and accept extension nodes documented per-capability — they are NOT free-form.

### Translation Boundary

Drivers translate `QueryFilter` to backend syntax. The API never inspects the translation result, but the API does enforce two invariants on input:

1. Every leaf node's `field_name` MUST resolve to a field in `stardust_fields`. Unknown fields are rejected at the API with `400 Bad Request` before the driver is invoked.
2. Every leaf node's `op` MUST be in the closed list above OR in a capability-declared extension list. Unsupported operators on the active driver are rejected at the API with `400 Bad Request` carrying a `capability_unsupported` discriminator.

Pre-flight validation (`is_filterable`) and schema-cache lookups operate on the resolved field references, not on raw strings. This is what allows ADR `0004` and ADR `0014` to remain operational across drivers — the validation has structural evidence to act on.

## Consequences

**Positive:**

- The API's pre-flight rejection layer (ADR `0004`) keeps working with arbitrary drivers. Pre-flight inspects the `QueryFilter` tree, not the driver's backend syntax.
- Structured-log events carry meaningful filter shape (`event: pre_flight_rejected`, `op: "in"`, `field: "status"`) regardless of driver. Dashboards do not need per-driver parsers.
- The schema-cache invalidation strategy (ADR `0015`) operates on field references, not driver-specific tokens — no driver swap requires cache redesign.
- The driver capability surface has a clean place to plug in: drivers declare which leaf operators they support, and the API rejects unsupported operators uniformly.
- Consumers do not need to know which driver is active. The same `QueryFilter` payload works against any driver that supports the requested capabilities.

**Negative:**

- Drivers cannot expose capabilities the `QueryFilter` schema does not model. Adding (e.g.) faceted aggregations requires extending the value object via ADR review, not a one-off driver-specific endpoint. This is intentional friction — preserving the consumer-facing contract.
- Translation cost is paid per request. For drivers with rich native query languages, the translation step may be non-trivial. Caching translated forms is a driver-internal optimization.
- The closed leaf-operator set bounds what consumers can express in v1. A consumer wanting (e.g.) a regex filter is rejected even if the active driver could trivially support it; the path is to add `regex` to the operator set via a future ADR.

**Rejected alternatives:**

- **Raw DSL passthrough** — defeats pre-flight rejection (the API has no structural view of the query), forces consumers to know the active driver, and silently breaks the structured-log event vocabulary. The alleged benefit ("expose every backend capability") is achievable through capability-extension nodes without sacrificing the consumer contract.
- **Separate endpoint per driver** (`/api/entries/mysql`, `/api/entries/meili`) — externalizes the driver choice into the URL space, defeating the entire point of a pluggable driver. Consumer code must branch on URL; deployments must coordinate consumer config with backend selection.
- **OpenAPI-style query-language synthesis** (e.g., GraphQL) — over-engineered for the matched-rows-with-cursor problem. The `QueryFilter` value object is a thin tree; full GraphQL adds parser, validator, executor, and tooling without a corresponding need.
- **Ad-hoc string DSL parsed centrally** (e.g., RHS-bracket syntax `?filter[status][in]=open,pending`) — a syntactic veneer over the same value object, but encoding rules vary across HTTP clients, URL encoders, and JSON shapes. The value object as a JSON body is unambiguous; the URL-DSL is not.

## Related

- ADR `0004` — Fail-fast on unindexed filters (pre-flight uses resolved field refs)
- ADR `0014` — Schema-level safety over runtime circuit breaking
- ADR `0015` — Database as sole daemon coordination point (cache invalidation)
- [`blueprints/search_driver_adapter.md`](../blueprints/search_driver_adapter.md) — closes Open Question #1
