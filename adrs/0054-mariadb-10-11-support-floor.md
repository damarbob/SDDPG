# 0054 - MariaDB 10.11+ Support Floor

**Status:** Proposed
**Created:** 2026-09-19

## Context

ADR [`0023`](0023-minimum-mysql-version.md) declared MySQL 8.0.13+ / Percona 8.0.13+ the supported floor and named MariaDB out of scope: "MariaDB's JSON storage representation and partial-index syntax differ enough that targeting it would be a separate ADR with its own compatibility matrix. ... A future MariaDB-targeted ADR could change that, but this ADR does not promise it." This is that ADR.

Three real MySQL-specific constructs stood between the engine and a second engine, identified by grepping the compiled SQL surface rather than reasoning from documentation:

1. The table-level `utf8mb4_0900_ai_ci` collation.
2. The ADR [`0017`](0017-schema-registry-as-coordination-contract.md) "at most one live slot per field" invariant, enforced by a functional unique index with a `CASE … END` expression.
3. The collation of a hypothetical `JSON_UNQUOTE(JSON_EXTRACT(...))` comparison against the ADR [`0013`](0013-json-payload-as-system-of-record.md) fallback payload.

Each was probed against real MariaDB containers — 10.6.28, 10.11.19, and 11.8.9 — paired against the project's own MySQL 8.0.13, rather than assumed from release notes. The candidate collation the earlier planning round had named, `utf8mb4_uca1400_ai_ci`, **does not exist on 10.6 or 10.11**; that collation family arrives only later than this LTS line. The functional unique index is rejected on all three MariaDB versions tested (errno 1064) — MariaDB has no equivalent syntax. `FOR UPDATE SKIP LOCKED` works identically on all three and was not a blocker.

The candidate that actually matters was not on the originally-considered list at all: the `nopad` collation family (`utf8mb4_general_nopad_ci`, `utf8mb4_unicode_nopad_ci`, `utf8mb4_unicode_520_nopad_ci`), present on both 10.6 and 10.11. MySQL's `0900_*` collations are inherently NO PAD and MySQL 8.0 ships no `nopad`-named collation at all, so **PAD SPACE vs NO PAD was the single largest source of disagreement, and it is pure configuration** — every PAD SPACE candidate loses four operators (`eq`, `neq`, `in`, `nin`) purely through trailing-space equality.

Built against the actual predicate shapes `Search\Mysql\SqlFilterCompiler` compiles — the twelve closed-v1 operators plus emoji equality, `ß`/`ss` equality, and both JSON-fallback comparisons, 17 cases total — `utf8mb4_unicode_520_nopad_ci` is the best available 10.x candidate. Decomposed to root causes rather than counted, the picture is:

| Root cause | 10.6 | 10.11 | 11.8 |
| :-- | :-- | :-- | :-- |
| Supplementary-plane sort boundary reversed vs. MySQL | present | present | absent (real `0900_ai_ci` available) |
| `JSON_UNQUOTE(JSON_EXTRACT(...))` returns `utf8mb3_general_ci` instead of `utf8mb4_bin` | present | absent | absent |

The supplementary-plane divergence: MySQL's `utf8mb4_0900_ai_ci` sorts supplementary-plane characters (e.g. most emoji) **before** all BMP text; MariaDB's UCA 5.20 sorts them **after**. A full `ORDER BY` comparison over a 33-value set confirmed relative order **within** the BMP is identical on both engines — case and accent equivalence classes, `Ω`, `中文`, `日本`, numeric strings, all in the same order. This is one boundary, not a general mismatch, and it is not a tie-order difference the mandatory `entry_data.id` tiebreak can rescue — it reorders genuinely unequal values. It surfaces only in the four range operators (`lt`, `lte`, `gt`, `gte`, and `between` when the range spans the boundary) and in an ADR [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) field sort, and only when the data contains supplementary-plane characters. Every equality-family operator (`eq`, `neq`, `in`, `nin`, `prefix`, `is_null`, `is_not_null`) agrees exactly.

The JSON-fallback divergence is independent and only affects 10.6: `JSON_UNQUOTE(JSON_EXTRACT(...))` returns `utf8mb3_general_ci` on 10.6 but `utf8mb4_bin` on 10.11 and on MySQL. On 10.6 this makes the ADR 0013 JSON payload fallback — the path a non-filterable field's read always goes through — compare **case-insensitively** where MySQL compares case-sensitively, in a **three-byte charset**. Unlike the sort-boundary divergence, this is not fixable by column collation; it would need the driver to emit an explicit `COLLATE utf8mb4_bin` on every JSON comparison, which is engine work rather than configuration, and no such comparison exists in shipped code today to carry it (`SqlFilterCompiler` cannot reach a non-filterable field — the ADR [`0004`](0004-fail-fast-on-unindexed-filters.md) pre-flight rejects it before compilation).

The functional unique index has a verified substitute on all three MariaDB versions: a `PERSISTENT` generated column holding the identical `CASE` expression, plus a plain `UNIQUE KEY` on it. It allows a second *tombstoned* slot for one field and refuses a second *live* one with SQLSTATE 23000, matching the MySQL functional index's observable behaviour exactly.

## Decision

**MariaDB 10.11 and later — but not 10.6 or earlier — is a supported second engine, with one documented behavioural divergence rather than an engineered-around one.**

1. **The floor is 10.11, not 10.6.** The JSON-fallback collation divergence found on 10.6 has no configuration-only fix and touches the non-filterable-field read path every deployment relies on; 10.11 does not have it. This mirrors ADR 0023's own reasoning for picking 8.0.13 over an older MySQL floor: pick the version that captures the needed behaviour without carrying a known-broken one.

2. **The chosen collation for MariaDB-target slot columns is `utf8mb4_unicode_520_nopad_ci`, set explicitly rather than inherited from the server default.** Both 10.6 and 10.11 default to `utf8mb4_general_ci`, the weakest of the three `nopad` candidates measured; a MariaDB-target deployment must not rely on the server default.

3. **The supplementary-plane sort-boundary divergence is accepted, not engineered around.** No collation available on the 10.x LTS line reproduces MySQL's `0900_ai_ci` ordering for supplementary-plane characters — that requires the `uca1400` family, which arrives only in later MariaDB releases past this floor. A MariaDB-target deployment's range operators (`lt`/`lte`/`gt`/`gte`/`between`) and ADR 0041 field sorts may therefore order rows containing supplementary-plane characters differently than the identical deployment would on MySQL. Every equality-family operator is unaffected and every deployment that never stores such characters is unaffected. See ADR 0041's Consequences for the sort-specific statement of this.

4. **The ADR 0017 live-slot invariant is enforced on a MariaDB target by the generated-column substitute**, not the functional unique index. `Support\Dialect` (added by the dialect-extraction work this ADR's Context section assumes) is where the per-engine DDL choice is made; this ADR does not itself specify the runtime mechanism that selects a target engine at bootstrap time, which is implementation work for the item that actually ships MariaDB DDL support.

5. **No engine-code change is needed for the JSON-fallback collation at this floor.** 10.11 already returns `utf8mb4_bin` off `JSON_UNQUOTE(JSON_EXTRACT(...))`, matching MySQL, so the divergence that motivated excluding 10.6 does not recur at 10.11+. `Support\Dialect`'s docblock, which names this construct with no method because no call site exists, is therefore correct to leave unchanged by this ADR — nothing ships a comparison for it to collate.

6. **This ADR authorises the compatibility target; it does not itself run the suite against it.** Running the full smoke suite green against MariaDB 10.11+, inverting or retiring the `mariadb-rejection` CI job, and building whatever bootstrap-time engine detection `Support\Dialect` needs are separate, larger pieces of work tracked outside this ADR.

## Consequences

**Positive:**

- The compatibility matrix stays a flat version floor per engine, mirroring ADR 0023: "MySQL 8.0.13+ / Percona 8.0.13+ or MariaDB 10.11+."
- Both the collation substitute and the unique-index substitute are empirically verified against real servers, not assumed from documentation — consistent with this project's standing rule that a load-bearing MySQL/MariaDB behavioural claim gets probed before it is designed around.
- The JSON-fallback divergence, the one substitute with no configuration-only fix, is avoided entirely by the floor choice rather than requiring engine code — no `SqlFilterCompiler` change is needed to reach this floor.

**Negative:**

- MariaDB 10.6 (and any earlier line) remains permanently unsupported under this decision, not merely unmeasured — the JSON-fallback collation divergence is a hard exclusion, not a gap to close later.
- A MariaDB-target deployment has one real, if narrow, cross-engine behavioural difference: range operators and field sorts order supplementary-plane characters oppositely from the same deployment on MySQL. This must be stated in consumer-facing deployment documentation, not only in this ADR.
- `EnvironmentTest::testServerIsMySql` can no longer be the sole floor gate once a MariaDB target is legitimate. Today the MariaDB rejection rests on two independent gates — the version-string check and `testPartialUniqueIndexSupported` (the functional-index probe). The generated-column substitute this ADR authorises removes the second gate's rejection power the moment it ships, leaving the version-string check to hold the line alone until the engine-detection work that follows this ADR inverts it deliberately.
- This ADR narrows ADR 0023's "MariaDB: not supported as a drop-in replacement" for the 10.11+ line specifically. ADR 0023's MySQL 8.0.13+ floor is otherwise unchanged.

## Related

- [ADR `0023`](0023-minimum-mysql-version.md) — The MySQL floor this ADR sits beside rather than replaces; 0023 explicitly reserved this decision for a future ADR.
- [ADR `0017`](0017-schema-registry-as-coordination-contract.md) — The live-slot invariant whose MariaDB substitute (generated column + plain UNIQUE) this ADR authorises.
- [ADR `0013`](0013-json-payload-as-system-of-record.md) — The JSON-fallback read path whose collation divergence on 10.6 is why the floor is 10.11, not 10.6.
- [ADR `0004`](0004-fail-fast-on-unindexed-filters.md) — Why no JSON-fallback SQL comparison exists in shipped code to need the `COLLATE utf8mb4_bin` fix in the first place.
- [ADR `0041`](0041-sort-ordering-and-the-anchored-cursor.md) — Gains the supplementary-plane sort-boundary consequence this ADR's floor makes concrete.
