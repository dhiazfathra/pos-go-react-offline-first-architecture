# ADR-0007: Device-prefixed local receipt sequences, server-assigned fiscal sequence

## Status
Proposed

## Date
2026-08-11

## Context

Receipts need numbers. Customers reference them, staff search by them, and many tax
jurisdictions require **gapless** per-store sequences for audit.

A gapless global sequence cannot be allocated offline. Two devices offline cannot agree on
"the next number" — the same impossibility as [ADR-0006](0006-oversell-accepted.md). And a
receipt must be printable at the moment of sale, so the number cannot be deferred until
sync.

This is easy to defer and expensive to retrofit: receipt numbers are printed, filed by
customers, referenced in support calls, and embedded in exported accounting data. Changing
the scheme after go-live is customer-visible.

## Decision

Two numbers, with different purposes.

**1. Device-prefixed local sequence — printed immediately.**

```
ST01-TAB03-000142
 │     │      └── monotonic, gapless per device, allocated offline
 │     └───────── device code
 └─────────────── store code
```

Allocated in the same IndexedDB transaction as the outbox append, so it is gapless per
device and survives restarts. This is what staff and customers see and search by.

**2. Server-assigned canonical fiscal sequence — assigned at ingest.**

Allocated in `received_at` order, gapless per store, only if the jurisdiction requires it.
Appears on reprints, fiscal reports and exports. Never printed at the till, because at
print time it does not yet exist.

Both are stored on the sale projection. Lookup works by either.

## Alternatives Considered

### Server-allocated number only
- Pros: single gapless sequence; simplest model.
- Cons: cannot print a receipt offline. Fatal.
- Rejected.

### Pre-allocated block of numbers per device
- Pros: gapless-looking global sequence; numbers available offline.
- Cons: unused numbers in a block become permanent gaps, which is exactly what the tax
  requirement forbids; blocks can exhaust mid-outage; requires online pre-allocation before
  an outage that was not scheduled.
- Rejected: fails the gapless requirement it exists to satisfy.

### UUID or timestamp-based receipt identifier
- Pros: trivially unique, no coordination.
- Cons: unreadable over the phone, unsearchable by humans, and not gapless — fails audit
  requirements in most jurisdictions.
- Rejected as the customer-facing number; the event UUID already covers internal identity.

### Device prefix only, no fiscal sequence
- Pros: one number, simpler.
- Cons: does not satisfy jurisdictions requiring a gapless per-store sequence.
- Rejected as a universal answer, but note that where no fiscal requirement exists, part 2
  is simply not enabled — the design degrades cleanly.

## Consequences

- Receipts print instantly, offline, with a stable human-usable number.
- Support and search must accept both formats and resolve either to one sale.
- The fiscal sequence is assigned in server-receipt order, which is **not** the order sales
  physically occurred. A device offline for two days receives fiscal numbers interleaved
  with other devices' later sales. Whether a jurisdiction accepts this must be verified —
  it is the single most likely place this decision breaks, and it is verified with a tax
  advisor, not by engineers.
- A device-local gap indicates data loss and is alertable, giving a second independent loss
  detector alongside `device_seq`.
- Device codes must be unique within a store, assigned at registration, and never reused —
  reuse would collide two devices' sequences.
- Exports and accounting integrations must state which number they use.

## Related
- [ADR-0005](0005-event-sourced-writes.md), [ADR-0006](0006-oversell-accepted.md)
- [Architecture §4.6](../architecture/04-sync.md#46-receipt-numbering)

## Open question carried
Whether the target jurisdictions accept a fiscal sequence assigned in receipt order rather
than occurrence order. Must be answered before `sales` is built.
