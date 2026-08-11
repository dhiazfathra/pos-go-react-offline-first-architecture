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

### Protecting Tier 1 on the device

Tier 1 puts customer, staff, and permission data on a tablet that leaves the back office,
so the replica is governed by four rules:

- **Minimisation is part of the policy, not an afterthought.** The policy proto declares a
  field allowlist per table, and the server projects to it before sending. Replicate the
  fields the till actually renders — for customers, identity and loyalty balance, not
  address, date of birth, or payment instrument; for staff, display name, PIN verifier, and
  role, never a password hash or contact details.
- **Encryption at rest is the platform's**, via full-disk encryption plus a device-lock
  requirement enforced by MDM. IndexedDB has no application-level encryption worth the
  name: any key the app can use offline is a key the app must store on the same device.
  This is stated as a limitation rather than papered over.
- **Revocation is time-bounded, not immediate.** Permission and staff changes reach a
  device on its next delta pull; there is no offline push. A device that has not synced
  within a configurable maximum-offline window stops accepting logins and prompts for
  connectivity, which bounds how long a revoked user can operate.
- **Device loss is handled by wipe, not by hope.** Every device is enrolled and remotely
  wipeable, and a device can be revoked server-side so it receives no further data and its
  sync credentials are rejected. Because remote wipe requires the device to be reachable,
  the maximum-offline window above is the actual guarantee.

Tier 3 is deliberately excluded from remote wipe as a first response — it is the only copy
of unsynced sales. Drain or export it before wiping where the device is still reachable;
where it is not, the loss is accepted and recorded.

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
- Eviction policy is explicit and ordered: Tier 2 oldest-first, then the evictable subset
  of Tier 1 — customer records outside recent activity, product images and other media,
  historical promo definitions — and **never** Tier 3.
- **Catalog, prices, tax rules, payment methods, staff, permissions, and store config are
  not evictable.** They are what makes a sale correct rather than merely recorded. If they
  cannot be held or are found missing at boot, the till **fails closed**: it refuses to
  open a shift and displays a "requires sync" state rather than trading on partial policy
  data. Discovering mid-shift that tax rules were evicted is a compliance incident; a till
  that will not open is a support call.
- Because required Tier 1 has a floor, that floor is a hardware-procurement number. It is
  measured against the largest supported catalog on the cheapest supported device, and a
  device that cannot hold it is not a supported till.
- `navigator.storage.persist()` is requested at install; quota is monitored and surfaced
  before it becomes critical.
- The replication policy being server-declared means the window can be tuned per tenant,
  per store, or per device role without a client release.

## Related

- [ADR-0003](0003-offline-scope.md), [ADR-0010](0010-pwa-not-native.md)
- [Architecture §4.2](../architecture/04-sync.md#42-replication-tiers)
