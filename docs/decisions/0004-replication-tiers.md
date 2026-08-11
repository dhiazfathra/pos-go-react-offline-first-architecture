# ADR-0004: Four replication tiers, scoped by access pattern

## Status
Proposed

## Date
2026-08-11

## Context

A device must hold enough data to trade through an outage. It cannot hold everything: POS
transactional volume is unbounded and growing, target hardware is cheap tablets, and
browser storage is quota-limited and evictable.

The naive local-first answer — "replicate the workspace into IndexedDB and hydrate it into
an observable graph" — works for a bounded document workspace and fails here for two
reasons:

1. Sales history grows without limit. A store doing a few thousand line items a day has
   millions of rows within a couple of years.
2. Hydrating everything into memory is correct only for data read on every keystroke. Most
   POS data is not.

The right sizing question is also not the obvious one. It is not "how much history fits in
memory" but **"what is the returns policy?"** — because the two real workflows needing
historical sales on-device are receipt reprint and return-against-sale, both point lookups.
Trading through a long outage needs room in the *forward* write queue, not depth of
history.

## Decision

Four tiers with different storage, sync, and durability rules.

| Tier | Contents | Scope | Storage | Sync |
|---|---|---|---|---|
| **1 — Hydrated** | Catalog, prices, promos, tax rules, payment methods, customers, staff, permissions, store config, feature flags | `store_id` + tenant-global | IndexedDB → **in memory** | Full replica, delta pull |
| **2 — Resident** | Sales, payments, stock movements, shifts | `store_id`, rolling window (default 35 d, set by returns policy) | IndexedDB, **queried by index, never hydrated** | Windowed pull, evictable oldest-first |
| **3 — Outbox** | Unsynced local writes | This device | IndexedDB, **never evicted** | Push-only, drained on server ack |
| **4 — Server-only** | Aggregates, audit log, cross-store analytics, history beyond the window | — | none | Online query, explicitly labelled in UI |

Two distinctions are load-bearing:

- **On-device ≠ in-memory.** Tier 1 is hydrated because it is read on every keystroke of a
  product search and every line-item price calculation. Tier 2 is not, because it is read
  twice a day by point lookup.
- **The outbox is a tier, not an implementation detail.** It is the only data in the
  system that exists nowhere else, and it gets guarantees nothing else gets.

The tier assignment per table is declared in a server-side replication policy
(`pos/sync/v1/policy.proto`), fetched by the client — not hardcoded in client code.

## Alternatives Considered

### One store, all history
- Pros: one predicate; no windowing logic; no "is this local?" branch anywhere.
- Cons: unbounded growth; hydration and storage both fail within a couple of years.
- Rejected: fails on the hardware this runs on.

### Whole tenant on every device
- Pros: managers see everything; simple mental model.
- Cons: at 20 stores it is the above times 20, and a cashier's tablet pays to store data
  the cashier never reads.
- Rejected: hydrating what the UI does not read.

### Dynamic per-client query subscriptions
- Pros: maximally flexible; scales to any access pattern; where mature sync engines end up.
- Cons: needs a query planner, server-side row-level filter evaluation, and subscription
  invalidation on write. Months of work before a single sale is rung up.
- Rejected for v1, but the policy proto is deliberately the seam that becomes this: swap a
  static predicate for a dynamic subscription and neither the tables nor the client change.

### Single time window across all tables
- Pros: one number to tune.
- Cons: applies a transactional-data concept to master data, where it makes no sense — a
  product created two years ago is still sold today.
- Rejected: the tiers exist because the access patterns genuinely differ.

## Consequences

- Storage is bounded and predictable; boot time is bounded by Tier 1, which is small and
  slow-changing.
- Tier 2's window is a business input. If returns run 90 days on a high-volume store, the
  IndexedDB budget must be re-measured against real hardware — a measurement, not an
  estimate.
- Tier 4 screens must be visibly online-only. A surprise spinner in the operating surface
  is an architecture violation; a labelled online-only report is a design choice.
- Eviction policy is explicit: Tier 2 oldest-first, then Tier 1 non-essential; **never**
  Tier 3.
- `navigator.storage.persist()` is requested at install; quota is monitored and surfaced
  before it becomes critical.
- The replication policy being server-declared means the window can be tuned per tenant,
  per store, or per device role without a client release.

## Related
- [ADR-0003](0003-offline-scope.md), [ADR-0010](0010-pwa-not-native.md)
- [Architecture §4.2](../architecture/04-sync.md#42-replication-tiers)
