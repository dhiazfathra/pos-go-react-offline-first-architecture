# ADR-0006: Offline oversell is accepted and reported, not prevented

## Status
Proposed — **requires explicit business sign-off**

## Date
2026-08-11

## Context

Two devices are offline. Both see one unit of a product in stock. Both sell it. Stock on
hand goes to −1.

The instinct is to prevent this. It cannot be prevented. Preventing it requires the two
devices to agree on who gets the last unit, and **agreement during a network partition is
precisely what a partition makes impossible.** This is not an engineering gap to be closed
with a better algorithm; it is CAP, and no amount of CRDT machinery changes it. A counter
CRDT converges to the correct sum of decrements — it does not stop the second decrement
from happening.

The only mechanism that guarantees no oversell is to refuse to sell when the device cannot
reach the server. For a POS, that means the till stops working during an outage, which is
the exact failure this entire architecture exists to prevent.

So the real question is not technical. It is: **which cost does the business prefer?**

## Decision

Accept oversell. Report it. Do not attempt to prevent it.

- Stock on hand is a server-side projection of the movement log
  ([ADR-0005](0005-event-sourced-writes.md)). Devices append movements; nobody writes a
  balance.
- A sale that drives stock negative is **accepted** by the server. The log correctly
  records that three units were sold when two existed. The server does not reject an event
  describing something that already physically happened.
- Local stock display is **advisory**: last-known balance plus this device's unsynced
  movements, labelled as approximate once the device has been offline beyond a threshold.
- Negative-stock conditions are surfaced daily as a reconciliation report, and immediately
  to a manager once the device reconnects.
- Soft, configurable warnings at the till ("last unit — stock may be inaccurate offline")
  reduce frequency without blocking a sale.
- Per-category policy is supported: a business may configure high-value or serialised
  items to require connectivity to sell. This is a narrow, opt-in exception, not the
  default.

## Alternatives Considered

### Refuse to sell when offline stock reaches zero
- Pros: no oversell.
- Cons: local stock is *already* stale after minutes offline, so this refuses legitimate
  sales of items that are physically present while still failing to prevent oversell
  between two devices. Worst of both.
- Rejected: costs revenue and does not achieve its goal.

### Reserve stock per device up front
- Pros: bounded oversell; each device knows what it may sell.
- Cons: requires online reservation before going offline (an outage is not scheduled);
  strands stock on the device that reserved it; reservation expiry during a long outage
  reintroduces the problem.
- Rejected: complexity without a guarantee. Worth revisiting only for a narrow class of
  high-value serialised goods.

### Counter CRDT for stock
- Pros: mathematically clean convergence.
- Cons: converges to the same negative number. Does not prevent anything. Adds a second
  data model and loses movement history.
- Rejected: solves a problem that is not the problem.

### Server-side stock lock with offline queue
- Pros: strict correctness when online.
- Cons: no mechanism at all when offline, which is the case under discussion.
- Rejected: does not address the scenario.

## Consequences

- Stores keep trading during outages. This is the point.
- Oversell will occur, and its business cost is real: backorders, substitutions, customer
  disappointment, and staff explaining a stockout. Frequency scales with outage duration
  and with how often stock genuinely runs to zero.
- Inventory accuracy is eventually consistent. Reports must show as-of times and must not
  present offline-period figures with false confidence.
- Support tickets of the form "it said 4 in stock" are expected. Contained by honest
  staleness labelling, not eliminated.
- **This is a business decision, not a technical default.** It must be signed off
  explicitly, and the mitigations — soft warnings, per-category thresholds, prompt
  reconciliation reporting — must be funded as product work rather than assumed.
- If a customer segment genuinely cannot tolerate oversell (serialised high-value goods,
  regulated items), the answer is the per-category online-required policy, applied
  narrowly.

## Related
- [ADR-0005](0005-event-sourced-writes.md), [ADR-0003](0003-offline-scope.md)
- [Architecture §4.4](../architecture/04-sync.md#44-conflict-resolution)
