# 0024 - Type Coercion Matrix for Retype Backfill

**Status:** Proposed
**Created:** 2026-05-06

## Context

When a field's declared type changes — or its `is_filterable` flag is promoted on a populated slot — ADR `0016` triggers the **sever → tombstone → assign → backfill → promote** lifecycle. During the backfill phase, the Reconciler reads the authoritative JSON payload at `entry_data.fields` and writes a value of the new declared type into the freshly-assigned slot column.

ADR `0016` pins the **failure behavior**: a value that "cannot be coerced" stores `NULL` in the new slot column, the JSON payload remains the system of record, and the row falls back to `JSON_EXTRACT` on retrieval (ADR `0013`). What ADR `0016` does **not** pin is the **predicate**: which JSON value / target-type pairs succeed, which produce `NULL`, and what canonical form a successful coercion produces in the slot column. Twelve type-pair semantics (4 × 4 minus the diagonal identity) sit unauthored. Without a fixed predicate, every implementer guesses — and the guesses carry architectural weight: `string → datetime` strict ISO 8601 vs lenient parsing, `numeric → int` truncate vs reject vs round, `int → datetime` epoch seconds vs millis vs reject, `numeric → string` decimal vs scientific notation, and so on.

A second, adjacent gap: ADR `0020` lists "Coercion-failure NULLs" as a silent-failure class operators must monitor (§Context, line 16) and references the event name `coercion_null` (line 88) — but the event is not declared in any daemon's vocabulary, and its payload sub-fields are unspecified. Without the event, the silent-NULL semantic becomes silently undiagnosable; the two gaps must close together.

The QueryFilter wire format (`blueprints/queryfilter_wire_format.md` §4.5) already pins authoritative typed-value rules for the four declared types (`string`, `int`, `numeric`, `datetime`) at the input boundary. Re-using those rules at the retype boundary — rather than authoring a second dialect — is both the principled choice and the simplest one.

## Decision

Three commitments govern Reconciler retype backfill coercion. The 4×4 type-pair matrix follows.

**Commitment 1 — Symmetry with QueryFilter §4.5.** A coercion succeeds if, and only if, the resulting value would validate as the target `declared_type` under the QueryFilter typed-value rules in `blueprints/queryfilter_wire_format.md` §4.5. The Reconciler does not maintain a second definition of "what is an `int` / `numeric` / `datetime`" — the wire-format rules are the single source of truth, applied at both ingress (caller wire input) and migration (retype backfill).

**Commitment 2 — Reject ambiguous epoch coercions.** The four cells `int → datetime`, `numeric → datetime`, `datetime → int`, `datetime → numeric` produce `NULL` with no parse attempt. Epoch interpretation (seconds? milliseconds? microseconds? UTC vs local time?) is a policy call that does not belong in engine semantics. Callers requiring epoch-style migration MUST bridge through a `string` intermediate field. Bridging via `string` forces the caller to author and version the epoch convention as data, not as engine behavior.

**Commitment 3 — Coercion failures are observable.** Every `NULL` produced by an attempted-but-failed coercion emits a structured-log event of name `coercion_null` per ADR `0020`, with a payload pinned in `blueprints/watcher_reconciler_daemons.md`. Coercion against a JSON value that was already absent or already JSON `null` — i.e., where no coercion was attempted — does NOT emit the event; only attempted-and-failed coercions do. This distinction is what makes the event actionable: a non-zero `coercion_null` rate over a backfill window indicates retype-incompatible data, not merely sparse fields.

### The Coercion Matrix

The diagonal is identity (`string → string`, `int → int`, etc.) and is omitted as a no-op. The twelve off-diagonal cells:

| from \ to | `string` | `int` | `numeric` | `datetime` |
|:---|:---|:---|:---|:---|
| `string` | — | base-10 integer literal: optional leading `-`, digits `[0-9]+`, no whitespace, no decimals, no thousands separators, no leading `+`; result MUST be in signed 64-bit range; else `NULL` | JSON-number grammar (RFC 8259 §6): optional leading `-`, integer or fractional, optional exponent `[eE][+-]?[0-9]+`; no whitespace; no `NaN`/`Infinity`; else `NULL` | strict RFC 3339 with explicit UTC offset (`Z` or `±HH:MM`); naive datetimes → `NULL`; precision up to `DATETIME(6)`; sub-microsecond truncated; result normalized to UTC (matching QueryFilter §4.5 §20) |
| `int` | decimal stringification, no leading zeros, leading `-` for negatives | — | identity (lossless widening; `int` ⊂ `numeric` representable range) | **`NULL` (rejected — Commitment 2)** |
| `numeric` | canonical decimal stringification: shortest representation that round-trips, no scientific notation, trailing zeros after the decimal trimmed, `-0` normalized to `0`, no trailing decimal point (e.g., `42.0` → `"42"`, `-0.0` → `"0"`, `1.5e2` → `"150"`) | only if integer-valued (`floor(v) == v`) AND in signed 64-bit range; otherwise `NULL` (no truncation, no rounding) | — | **`NULL` (rejected — Commitment 2)** |
| `datetime` | RFC 3339 normalized to UTC, with `Z` suffix (not `+00:00`); microsecond precision preserved; trailing zeros in fractional seconds NOT trimmed (full `.000000` is canonical) | **`NULL` (rejected — Commitment 2)** | **`NULL` (rejected — Commitment 2)** | — |

**Notes on matrix application:**

- The matrix applies to the Reconciler retype backfill path (per ADR `0016` §Decision and the `backfilling` slot status of ADR `0017`) only. It does **not** apply to wire-format input validation — that remains the strict QueryFilter §4.5 rule set, where the input MUST already be in the target type.
- A JSON value that is `null` in the payload remains `NULL` in the slot column and does NOT emit `coercion_null` (no coercion was attempted).
- A JSON key that is absent from the payload yields `NULL` in the slot column and does NOT emit `coercion_null` (no coercion was attempted).
- A JSON value whose JSON type is structurally incompatible with the source declared type (e.g., the source is `string` but the JSON value is an object or array) is treated as a coercion failure: store `NULL`, emit `coercion_null` with reason `unparseable`. This case should be rare in practice — a well-formed payload submitted under the previous `declared_type` should already match its JSON type — but it can arise from out-of-band JSON edits or from migrations that change source type and target type in successive operations.
- Coercion is per-row and independent. A failure on one row never aborts the chunk (per ADR `0018`: coercion failures are explicitly NOT poison pills) and never delays Reconciler progress.

### Observability

The `coercion_null` event payload is normative; full sub-field documentation lives in `blueprints/watcher_reconciler_daemons.md` §Reconciler event vocabulary. The event MUST be emitted at level `warn` (not `error`): the row is preserved authoritatively in JSON per ADR `0013`, so the situation is operationally surprising but not data-loss. Event reasons include `out_of_range`, `non_integer`, `malformed_datetime`, `malformed_number`, `epoch_coercion_rejected`, and `unparseable` (closed taxonomy; see the blueprint).

Operators MUST monitor `coercion_null` event volume per active backfill (correlated by `slot_assignment_id`). A non-zero rate across a meaningful sample of the tenant/model partition is the signal that retype-incompatible data is being silently NULL'd; ADR `0016` already commits to the silent-NULL behavior, this ADR makes that behavior auditable.

## Consequences

**Positive:**

- A single definition of "what is an `int` / `numeric` / `datetime`" is shared across QueryFilter §4.5 (caller wire input) and the Reconciler (retype backfill). Implementations cannot diverge between the two boundaries.
- Twelve previously-implicit type-pair decisions are now explicit and reviewable. A future implementer reaches the matrix without authoring it.
- Ambiguous epoch coercions are explicitly out of scope, preventing silent locale/precision bugs of the form "this implementation interpreted the int as seconds, that one as milliseconds, and the data drifted by ~1000×".
- The `coercion_null` event closes ADR `0020`'s observability commitment for coercion-failure NULLs. The silent-NULL semantic of ADR `0016` is now diagnosable from the structured log.
- The matrix's strictness is symmetric with the QueryFilter wire-format strictness, which means callers who already submit data validating against §4.5 will see zero `coercion_null` events for fields whose value has not been edited out-of-band.

**Negative:**

- Callers who relied on lenient parsing for retype migration (e.g., expecting `"42.0"` to coerce `string → int`) will see those rows silently NULL'd in the slot column. The data is preserved in the JSON payload (ADR `0013`), but their indexed view will be incomplete until the JSON values are normalized. This is mitigated by the `coercion_null` event making the situation visible.
- `string → numeric` admits scientific notation (per the JSON-number grammar inherited from §4.5 §19). Audits expecting a "plain decimals only" rule will need to read the matrix carefully; the ADR explicitly allows `"4.2e1"`.
- The `numeric → string` canonical stringification rule requires implementation discipline. A naive `(string)$double` cast in PHP will produce locale-dependent output and may emit scientific notation for large or small values. The matrix's canonical form must be implemented explicitly, not delegated to the platform default.
- Callers needing epoch-based int↔datetime migration must bridge through `string` (e.g., retype `int → string`, then `string → datetime` after normalizing the JSON payload to RFC 3339). This is a two-retype operation rather than one. The complexity is the price of refusing to bake epoch policy into the engine.

**Rejected alternatives:**

- **Defer to MySQL `CAST` semantics.** MySQL `CAST` has surprising and version-sensitive behavior (`CAST('42abc' AS UNSIGNED)` returns `42` with a warning, `CAST('abc' AS DATETIME)` returns `NULL` but with locale-dependent edge cases). Delegating coercion to the database would couple StarDust's data semantics to MySQL version drift. Explicit application-side coercion using §4.5 rules is deterministic and version-independent, satisfying ADR `0023`'s minimum-version posture without inheriting CAST's edge cases.
- **Interpret int/numeric as Unix epoch seconds.** Selects one convention from many (seconds vs millis vs micros, UTC vs local) and bakes it into engine semantics. Callers whose data uses a different convention would see catastrophic drift (~1000× for seconds-vs-millis). Bridging through `string` keeps the convention as caller-controlled data.
- **Per-field configurable lenient parsing.** Adding a "lenient: true" flag to the field metadata explodes the contract surface (twelve cells × two modes = twenty-four behaviors), the test matrix, and the reviewer cognitive load. The strict-only path is a forcing function for caller-side data hygiene before retype, which is the correct ordering of concerns.
- **Fail the retype on first uncoercible row.** Rejected by ADR `0016` already (the lifecycle commits to silent-NULL with JSON fallback). This ADR fixes the predicate, not the failure behavior.
- **Emit a separate event per reason (e.g., `coercion_out_of_range`, `coercion_malformed_datetime`).** Multiplies the event vocabulary without operational benefit; the `reason` sub-field on a single `coercion_null` event provides the same triage capability with one dashboard query.

## Related

- ADR `0013` — JSON Payload as System of Record (why coercion failures are recoverable).
- ADR `0016` — Field Type Change Lifecycle (commits to silent-NULL on coercion failure; this ADR pins the predicate).
- ADR `0017` — Schema Registry as Coordination Contract (the `backfilling` slot status under which this matrix applies).
- ADR `0018` — Reconciler Poison-Pill Semantics (explicitly: coercion failures are NOT poison pills).
- ADR `0020` — Structured JSON Logging Mandate (event-vocabulary discipline; declares `coercion_null` as a known silent-failure-class signal).
- `blueprints/queryfilter_wire_format.md` §4.5 — the typed-value rules this ADR mirrors at the retype boundary.
- `blueprints/watcher_reconciler_daemons.md` — declares the `coercion_null` event and its sub-field payload.
