# ADR-0003: Offline scope — operating surface fully offline, administrative surface online-only

## Status
Proposed

## Date
2026-08-11

## Context

"The whole app works offline" is the obvious requirement for a POS, and it is the wrong
requirement stated at the wrong granularity. POS entities are not uniform:

- **Revenue-critical writes** — sale, refund, payment, cash movement, shift open/close,
  stock count, price override. Offline capability here is the reason the product exists.
- **Reads needed to make those writes** — catalog, price, promo, tax, customer, staff,
  approximate stock.
- **Administrative writes** — create product, change a price list, edit a tax rule, add a
  user, change permissions, edit store config. Low-frequency, single-writer in practice,
  performed at head office, high blast radius when wrong.

Making the third group offline-writable adds conflict handling to every master-data
entity, makes permission changes reconcilable-after-the-fact, and creates the possibility
of a tax rule merged wrong — a compliance problem — to save a manager an inconvenience
they will encounter approximately never.

## Decision

Draw the line by surface, not by entity:

> **Offline-complete for the operating surface. Online-required for the administrative
> surface.**

- Every screen a cashier, shift supervisor, or floor manager touches during a trading day
  works fully offline, writes included.
- Back-office authoring requires connectivity and says so explicitly in the UI.
- In the client, `app/routes/_admin/` is a hard boundary: its layout route checks
  connectivity and renders a designed offline state rather than a broken form.

This is a **policy** line, not a technical wall. The sync engine is capable of carrying
admin writes; the restriction is a deliberate reduction of conflict surface.

## Alternatives Considered

### Everything offline, including administration
- Pros: no surprises for the user; one uniform rule; no online/offline split to explain.
- Cons: every master-data entity needs conflict resolution and a reconciliation UI;
  permission and tax-rule conflicts carry compliance consequences; substantially larger
  build and test surface for near-zero usage.
- Rejected: cost is concentrated exactly where the benefit is smallest.

### Only the register works offline
- Pros: smallest possible offline surface.
- Cons: a shift cannot be closed, a return cannot be processed, and a stock count cannot
  be recorded during an outage — all of which are normal trading-day activities.
- Rejected: too narrow to deliver business continuity.

### Offline reads everywhere, offline writes only for sales
- Pros: simple to state.
- Cons: same gap as above — refunds, cash movements and shift close are operationally
  essential and would break.
- Rejected: the write set is drawn from a technical instinct rather than from how a store
  actually operates.

## Consequences

- Roughly 60% of the conflict surface disappears for approximately zero operational loss.
- Master-data conflicts reduce to concurrent-online-admin edits: ordinary optimistic
  concurrency (version column, 409, reload), not a distributed merge problem.
- Someone will eventually ask "why can't I add a product offline?" The answer is a
  business conversation. Expect it, and answer it honestly rather than as a technical
  limitation.
- The `_admin` boundary must be genuinely designed — an explicit, informative offline
  state — or users will experience the policy as a bug.
- Relaxing this later is additive and does not invalidate anything built.

## Related
- [ADR-0004](0004-replication-tiers.md), [ADR-0005](0005-event-sourced-writes.md)
- [Architecture §4.1](../architecture/04-sync.md#41-what-offline-means-here)
