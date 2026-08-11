# ADR-0005: Event-sourced operational writes, LWW for master data

## Status
Proposed

## Date
2026-08-11

## Context

Full-offline writes across every operating screen appear to require a general conflict
resolution strategy — LWW, CRDTs, or manual reconciliation — applied uniformly.

That framing is wrong for POS, and correcting it is the most consequential decision in
this architecture.

**A completed sale is not mutable state. It is an immutable fact with a timestamp.** Two
devices cannot "conflict" on a sale any more than two cash registers conflict by both
taking money. They produce two distinct events. There is no merge function because there
is nothing to merge.

This applies to the overwhelming majority of POS write volume: sales, payments, refunds,
cash movements, stock movements. Once seen, the conflict problem shrinks to a small set of
slow-changing master-data tables — most of which are online-only anyway per
[ADR-0003](0003-offline-scope.md).

The remaining trap is stock on hand. It looks like a counter two devices decrement, so it
looks like it needs a counter CRDT.

## Decision

Two write models, explicitly separated.

**Operational data — event-sourced.**

- The client appends immutable domain events to a durable outbox: `SaleCompleted`,
  `RefundIssued`, `PaymentTaken`, `StockMoved`, `ShiftOpened`, `CashMoved`.
- Each event carries: client-generated UUID v7 (idempotency key), device ID, monotonic
  gapless `device_seq`, `occurred_at` (device clock, recorded but untrusted), and a
  Protobuf payload.
- The server validates, inserts into an append-only `domain_events` table
  (`ON CONFLICT DO NOTHING` on `event_id`), and updates relational projections **in the
  same transaction**.
- Projections are what REST/gRPC reads serve. Rebuilding a projection from the log is a
  supported, tested operation.
- Event tables are `REVOKE UPDATE, DELETE` for the application role.

**Master data — ordinary mutable rows.**

- LWW per field with a `version` column; online-only per ADR-0003, so in practice this is
  optimistic concurrency: 409 on version mismatch, client reloads.

**Stock on hand is not a writable field.**

```
stock_on_hand(product, store) = Σ movements(product, store)
```

Devices append movements. Nobody writes a balance. See
[ADR-0006](0006-oversell-accepted.md) for the consequence.

## Alternatives Considered

### Server-authoritative LWW per field, everywhere
- Pros: simplest; no client CRDT library; one uniform rule.
- Cons: applying LWW to a sale means two simultaneous sales overwrite each other. Silently
  loses money.
- Rejected: category error — sales are facts, not values.

### CRDTs everywhere (Yjs / Automerge / Loro)
- Pros: genuine convergence, no lost writes, mature libraries.
- Cons: CRDT state must live in Postgres alongside relational rows; the Go side must
  understand the CRDT format; Protobuf stops being the source of truth; and it still does
  not answer "was this sale valid?" because CRDTs converge without validating.
- Rejected: solves convergence, which is not the problem. The problem is durable,
  auditable, validated financial facts.

### Counter CRDT for stock specifically
- Pros: stock converges correctly without a movement log.
- Cons: converges to the correct *sum of decrements*, which is not the same as preventing
  oversell — that requires consensus, which is unavailable during a partition. Adds a
  second data model for one field, and loses the movement history that inventory
  management actually needs.
- Rejected: a movement log is strictly more useful and structurally simpler.

### Full event sourcing including master data
- Pros: one uniform model; complete audit of everything.
- Cons: master-data reads become projection rebuilds for no benefit; CRUD admin screens
  become substantially more work.
- Rejected: event sourcing where it pays, CRUD where it does not.

## Consequences

- ~95% of write volume has no conflict resolution at all, by construction.
- Retry-on-timeout is safe: client-generated `event_id` makes re-push a no-op. Given that
  lost acks are the *normal* condition on the target networks, this is the single most
  important property in the protocol.
- `UNIQUE (device_id, device_seq)` makes data loss detectable — a sequence gap is
  alertable, not discovered at month-end.
- `occurred_at` (device) and `received_at` (server) are both stored and never conflated.
  Device clocks drift and are user-settable; correctness arguments use `received_at`,
  staff-facing reports use `occurred_at`.
- **Two write models means developers must know which one they are in.** Contained by
  visibly different packages, different repository base types, and database-level
  permissions that make the wrong instinct fail loudly.
- Reporting queries hit projections, not the event log.
- Protobuf payloads are opaque to ad-hoc SQL. Mitigated by projections and a
  `pos-events decode` CLI, which should exist before the first incident.
- Synchronous projections cost some write latency. At POS volumes this is not close to a
  problem, and it avoids the far worse "the number was wrong for four seconds" class of
  bug in a money product.
- Corrections are compensating events, never updates or deletes. A void is an event; a
  refund is an event. History is never rewritten.

## Related
- [ADR-0003](0003-offline-scope.md), [ADR-0006](0006-oversell-accepted.md),
  [ADR-0007](0007-receipt-numbering.md), [ADR-0002](0002-protobuf-buf-contracts.md)
- [Architecture §4.3–4.4](../architecture/04-sync.md#43-the-offline-write-path)
