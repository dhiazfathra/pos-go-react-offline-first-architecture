# Design session: POS monorepo architecture

**Date:** 2026-08-11
**Outcome:** [`docs/architecture/`](../../architecture/) and [`docs/decisions/`](../../decisions/)

This records the decision path. The design itself lives in the architecture docs and ADRs
rather than being duplicated here.

## Scope

Design the monorepo architecture for a multi-tenant, multi-store POS platform: Go
modular monolith, Protobuf/Buf contracts, React/TanStack Start local-first PWA, PostgreSQL
source of truth. Stack constraints were given and treated as fixed; deviations would have
been raised as open questions. None were needed.

## Decision path

Three questions were resolved in sequence. The second answer to each is the one that was
recorded — the first pass reasoned from a general local-first playbook, which turned out to
be solving a different problem.

### Q1 — Offline scope

- *First answer:* whole app offline, reads and writes.
- *Revised:* **operating surface fully offline, administrative surface online-only.**
- *Why it changed:* POS entities are not uniform. Making back-office authoring (tax rules,
  permissions, product creation) offline-writable adds conflict handling with compliance
  consequences to save a manager an inconvenience they encounter approximately never. It
  removes roughly 60% of the conflict surface for near-zero operational loss.
- → [ADR-0003](../../decisions/0003-offline-scope.md)

### Q2 — Replica scope per device

- *First answer:* one store + a 90-day window, sized to keep the hydrated graph small.
- *Revised:* **four tiers scoped by access pattern**, with the history window set by the
  **returns policy** rather than by memory budget.
- *Why it changed:* the original constraint was borrowed — "keep the observable graph small
  so boot is fast" is a document-app concern. The POS constraints are: trade through the
  longest plausible outage (needs outbox room, not history depth), support receipt reprint
  and returns (point lookups, not scans), and fit cheap tablet storage. That separates
  *on-device* from *in-memory*, and promotes the outbox to a first-class tier.
- → [ADR-0004](../../decisions/0004-replication-tiers.md)

### Q3 — Conflict resolution

- *First answer:* split by entity class — events for operational data, LWW for master data.
- *Revised:* same conclusion, but reframed from a convenience into the central insight —
  **a completed sale is an immutable fact, not mutable state**, so ~95% of write volume has
  no conflict by construction. Stock on hand is not a writable field; it is a projection of
  the movement log. Offline oversell is a business event, not a data conflict, and cannot
  be prevented during a partition.
- → [ADR-0005](../../decisions/0005-event-sourced-writes.md),
  [ADR-0006](../../decisions/0006-oversell-accepted.md),
  [ADR-0007](../../decisions/0007-receipt-numbering.md)

## The reframe

The user's challenge — *"linear-local-first-architecture might solve a different problem
than what POS faces"* — was correct and changed all three answers.

A collaborative document app optimises *perceived latency*; convergence to any consistent
state is fine. A POS optimises *business continuity and auditability of money*; some
conflicts have a correct answer defined by accounting, not by a merge function. Offline
duration differs by orders of magnitude, and the data is an unbounded append-only financial
stream rather than a bounded workspace.

Resulting framing, which drives the whole design:

> **POS is an offline event-sourcing problem with a cached read model on the side.**

What still transfers from the local-first playbook: the client tactics (optimistic local
writes, granular reactivity, local search, code splitting, service-worker precache,
composited-only animation). On a till these are throughput, not polish.

What does not: the architecture premise. The server validates rather than merges, Postgres
stays the ledger of record, and the client is a scoped cache plus a durable outbox rather
than a peer replica.

## Not decided here

Eleven blocking questions are collected in
[`docs/README.md`](../../README.md#blocking-questions). The two that could most change the
design are payment-terminal offline capability and jurisdictional fiscal requirements.
