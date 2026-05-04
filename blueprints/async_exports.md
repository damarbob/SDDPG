# Blueprint: Asynchronous Exports — Relocated to StarGate

> **Status:** Relocated 2026-05-04
> **Original Author:** Damar Syah Maulana
> **Original Created:** 2026-04-10

## Notice

This blueprint's content has been relocated. It defined the `/api/exports` HTTP endpoint contract: request payload shape, `202 Accepted` response, job-status polling, artifact download, and consumer-facing acceptance criteria. Under the StarDust/StarGate architectural split, HTTP endpoints and consumer-facing wire protocols are owned by StarGate, not StarDust.

## Engine residue

The Chronicler daemon — claim semantics, bounded paginated probe loop, file-streaming, observability — is engine-side and remains in StarDust. It is captured in [`chronicler_daemon.md`](chronicler_daemon.md), which inherits the daemon decisions made in this blueprint without the HTTP framing.

## Related

- [`chronicler_daemon.md`](chronicler_daemon.md) — engine-side daemon blueprint.
- [`../adrs/0010-asynchronous-exports.md`](../adrs/0010-asynchronous-exports.md) — companion ADR (also relocated; engine residue note in place).
