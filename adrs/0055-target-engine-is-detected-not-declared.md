# 0055 - The Target Engine Is Detected, Not Declared

**Status:** Accepted
**Created:** 2026-09-20

## Context

ADR [`0054`](0054-mariadb-10-11-support-floor.md) made MariaDB 10.11+ a supported second engine and deliberately stopped short of one thing, in its own §4: "this ADR does not itself specify the runtime mechanism that selects a target engine at bootstrap time, which is implementation work for the item that actually ships MariaDB DDL support." This is that decision.

`Support\Dialect` already exists, extracted ahead of this work. It names the two constructs that diverge — the table-level collation (`utf8mb4_0900_ai_ci` on MySQL, `utf8mb4_unicode_520_nopad_ci` on MariaDB per 0054 §2) and the ADR [`0017`](0017-schema-registry-as-coordination-contract.md) live-slot uniqueness mechanism (a functional unique index on MySQL, a persisted generated column plus a plain `UNIQUE` on MariaDB per 0054 §4) — but every method returns MySQL syntax unconditionally. It is a seam with no switch, deliberately: the extraction work built the seam and left the switch to this ADR.

Something must therefore answer "which engine is this?" before the first `CREATE TABLE` is emitted, and today nothing can. **No production code path has ever inspected the server's identity or version.** `EnvironmentTest` reads `VERSION()`, but that is a test; in the engine itself the DDL has simply succeeded or failed, which was a complete strategy for exactly as long as there was one supported target.

Three mechanisms were considered:

| Option | Wrong DDL possible? | Permanent `Config` surface | Escape hatch |
| :-- | :-- | :-- | :-- |
| **A. Detect from the connection** | No — the server is the authority, and a mismatch is not expressible | None | None |
| **B. `Config` declares the target** | **Yes, silently, in one direction** | +1 field | Is the hatch |
| **C. Detect, with a `Config` override verified against detection** | No | +1 field | Yes, but inert when needed |

**The asymmetry in B is what decides this ADR.** Declaring *MySQL* against a MariaDB server fails loudly at the first `CREATE TABLE` — errno 1273, an unknown collation. Declaring *MariaDB* against a MySQL server emits a persisted generated column plus a plain `UNIQUE`, which **works on MySQL**: it enforces the 0017 invariant correctly, passes every test, and quietly establishes a third permanent registry shape that diverges from the documented one. That is the failure mode ADR [`0043`](0043-pages-provision-only-indexed-columns.md) already cost this project once — correct-looking output over wrong underlying state, found long after the change that caused it.

C closes that hole, but its override is inert in the one scenario that motivates having an override at all: if detection misreads the server, the agreement check misreads it identically, and the override cannot be used to correct what it is being validated against.

## Decision

**The target engine is detected from the live connection. There is no configuration input, and a server the engine does not recognise is refused rather than guessed at.**

1. **`Support\ServerEngine` is a closed enum with two cases: `MYSQL` and `MARIADB`.** Percona is `MYSQL`, not a third case — ADR [`0023`](0023-minimum-mysql-version.md) has always treated Percona 8.0.13+ as the same target, it reports a MySQL version string, and it needs byte-identical DDL. The enum names dialects, not products.

2. **`Dialect` stays `final`, non-instantiable and static; each method gains a `ServerEngine` parameter.** Not an interface with two implementations. Keeping both engines' literals adjacent in one file is what makes "what does MariaDB do for this construct?" answerable by opening one file rather than two, it keeps `tests/Smoke/DialectTest.php` a DB-free scan over pure functions, and it preserves the shape the extraction work deliberately mirrored from `Slot\IndexedSlotPredicate`. The cost is two constructor parameters: `Bootstrap\Bootstrapper` and `Page\PageProvisioner` are the only production callers.

3. **The detected engine is memoised on `StarDust`, on the `lockNamespace()` precedent (ADR [`0053`](0053-advisory-lock-names-are-qualified-per-installation.md) §7), and it has to live there rather than in `bootstrap()`.** `PageProvisioner` emits DDL at Watcher time — arbitrarily later, in a different process — so a value resolved as a local in `bootstrap()` would not reach the second DDL emitter at all. It is not a `Config` field for the same reason `Config::$lockNamespace` resolves outside `Config`: it is a fact about the connection, and `Config`'s constructor performs no I/O of any kind.

4. **Detection fails closed.** An unrecognised server, or a recognised one below its floor (MySQL/Percona 8.0.13, MariaDB 10.11), throws rather than defaulting to MySQL. For MariaDB this is load-bearing rather than hygiene: 10.6 is a *hard* exclusion under 0054 §1, and once the generated-column substitute ships **nothing in the DDL rejects it** — the collation exists there, the substitute works there, and the deployment would come up and run with the ADR [`0013`](0013-json-payload-as-system-of-record.md) fallback comparing case-insensitively in a three-byte charset. The DDL stopped being the floor gate the moment the substitute landed; this clause is what replaces it.

5. **The engine is matched on the `MariaDB` marker in the version string, never on a version-number range.** MariaDB has long reported `5.5.5-10.11.19-MariaDB` for old-client compatibility, and a naive leading-triple parse reads that as `5.5.5` — which fails closed, but as "MySQL below floor", with an error message that sends an operator hunting a problem they do not have. Match the marker first; parse the version only after stripping a legacy prefix.

6. **The version source must be probed before it is fixed.** `PDO::ATTR_SERVER_VERSION` arrives on the connection handshake at no round-trip cost and is the preferred source, but whether it carries the `5.5.5-` prefix is client-library behaviour. **This is unmeasured at the time of writing** and is called out here rather than assumed, per this project's standing rule that a load-bearing MySQL/MariaDB behavioural claim is probed against real servers. If the shape does not hold uniformly across the supported PHP matrix and both client libraries, fall back to `SELECT VERSION()`; one round trip at bootstrap is not worth defending.

7. **There is no `Config` override, and the omission is a decision rather than an oversight.** An override would serve a deployment behind a proxy that rewrites the handshake version, or an unrecognised MySQL-compatible fork. No such deployment exists today, and this project does not build seams for phases that have not arrived. `Config`'s constructor is strictly append-only, so adding the field the day a real deployment needs one is a non-breaking change — which is precisely the argument against adding it speculatively now.

8. **The detected engine is publicly readable: `StarDust::serverEngine(): ServerEngine`.** This is not decoration. ADR 0054 §3 accepts a real behavioural divergence rather than engineering around it — range operators and ADR [`0041`](0041-sort-ordering-and-the-anchored-cursor.md) field sorts order supplementary-plane characters at opposite ends on the two engines — and a consumer otherwise has no way to determine which side of that divergence a given deployment sits on.

### New events

None. `src/Bootstrap/` emits no events today, and this ADR does not make it the first emitter; ADR [`0020`](0020-structured-logging-mandate.md)'s vocabulary is untouched. That restraint has a second motive beyond proportionality: the vocabulary guard scans a fixed list of directories and returns an empty set for one it does not scan, so a first event in `src/Bootstrap/` would also need its own scan method or it would go unenforced while the suite stayed green. The §8 accessor is the observability surface instead.

### Considered and rejected: feature detection

Probing capability rather than identity — attempt the functional unique index, catch errno 1064, conclude MariaDB — is the more honest mechanism in principle. It is rejected for three reasons, the third decisive:

1. Two divergent constructs means two probes, and every construct that diverges later adds another. Identity is probed once.
2. The probes are DDL, and `bootstrap()` is idempotent and re-run routinely by design. A create-and-drop on every invocation is churn, and leaves debris behind a failed run.
3. **The 10.6 exclusion is not a capability question at all.** MariaDB 10.6 accepts every construct the engine needs; what disqualifies it is the *collation* `JSON_UNQUOTE(JSON_EXTRACT(...))` returns. No cheap feature probe answers that, so a capability-only mechanism would cheerfully bootstrap the one version 0054 went out of its way to exclude.

## Consequences

**Positive:**

- **The silently-wrong configuration becomes unrepresentable.** The MariaDB-on-MySQL direction described in Context — valid DDL, wrong invariant mechanism, permanent divergent registry shape — cannot be reached, because nothing can assert an engine over the server's own answer.
- **Existing MySQL deployments need no change and gain no configuration.** Every consumer keeps constructing `Config` exactly as before, and the second engine costs them nothing to ignore.
- **`Dialect` stays a pure function of (construct, engine).** Both branches are therefore covered by a DB-free source scan, which neither a config-driven nor an interface-based design gives for free — a config-driven one moves the branch out of the class under test, and an interface-based one turns "do not re-inline the literal" from a scan into a convention.
- **The version floor stops being documentation and becomes enforcement**, on both engines, for the first time.

**Negative:**

- **The engine gains a runtime version gate it has never had.** A deployment on an unrecognised MySQL-compatible fork that works today — because it accepts the DDL — will refuse to bootstrap after this. That is a deliberate breaking change for such a deployment, taken while the tag is `0.3.0-alpha.1` and no compatibility promise is outstanding. It is the direct and intended cost of §4.
- **There is no escape hatch.** An operator behind a proxy that rewrites the handshake version has no recourse but to connect around it. This is the accepted cost of §7, and the first ADR to revisit it will be the one that meets such a deployment.
- **Detection trusts a version string, which is weaker authority than behaviour.** The rejected alternative is the stronger mechanism considered in isolation; this ADR picks identity over capability because the floor decision turns on a collation property that capability probing cannot see.
- **One more thing must now be right for the DDL to be right.** Before this, MySQL-specific DDL was unconditionally correct because there was one target; the failure mode "correct DDL, wrong engine" did not exist. It exists now, and §5 exists because the most likely way to reach it is a version string that does not parse the way it looks like it should.

## Related

- [ADR `0054`](0054-mariadb-10-11-support-floor.md) — Authorises MariaDB 10.11+ as a target and explicitly defers this mechanism in its §4. This ADR completes that deferral; it does not amend or narrow anything 0054 decided.
- [ADR `0023`](0023-minimum-mysql-version.md) — The MySQL 8.0.13 floor, and the Percona equivalence that makes §1's enum two cases rather than three.
- [ADR `0017`](0017-schema-registry-as-coordination-contract.md) — The live-slot invariant whose enforcement mechanism becomes per-engine, and therefore the reason a wrong detection is a correctness problem rather than a cosmetic one.
- [ADR `0013`](0013-json-payload-as-system-of-record.md) — The JSON-fallback read path whose collation on 10.6 is what makes §4's floor check load-bearing rather than hygiene.
- [ADR `0053`](0053-advisory-lock-names-are-qualified-per-installation.md) — `StarDust::lockNamespace()`, the precedent for memoising a connection-derived fact on the entry point rather than storing it on `Config`.
- [ADR `0026`](0026-framework-neutral-composer-packaging.md) — Framework-neutral packaging makes an injected `PDO` the engine's only database handle, which is why detection reads the live connection rather than parsing a DSN.
- [ADR `0041`](0041-sort-ordering-and-the-anchored-cursor.md) — The field sort that carries 0054's supplementary-plane divergence, and the reason §8 makes the engine publicly readable.
