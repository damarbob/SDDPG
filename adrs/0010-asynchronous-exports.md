# 0010 - Asynchronous Exports — Relocated to StarGate

**Status:** Relocated 2026-05-04
**Created:** 2026-04-16

## Notice

This ADR's content has been relocated. It defined an HTTP endpoint contract (`POST /api/exports`, `202 Accepted` response, job polling, format negotiation, artifact download) for unbounded data retrieval. Under the StarDust/StarGate architectural split, HTTP endpoints and consumer-facing wire protocols are owned by StarGate, not StarDust.

The ADR slot (0010) is retained here so the numeric sequence is uninterrupted, per the project's append-only ADR convention. ADRs are not renumbered; this notice is the supersession marker.

## Engine residue

The decision to introduce **The Chronicler** — an independent background PHP daemon that materializes export jobs by consuming the bounded read path — is engine-side and remains in StarDust. The daemon's lifecycle, claim semantics, and observability surface are captured in [`blueprints/chronicler_daemon.md`](../blueprints/chronicler_daemon.md). The daemon's _trigger_ (job submission via HTTP endpoint) belongs to StarGate.

## Related

- [`blueprints/chronicler_daemon.md`](../blueprints/chronicler_daemon.md) — engine-side daemon blueprint.
