# 4. Local-First Sync Strategy

The load-bearing section. If this is wrong, the product does not work.

## 4.1 What "offline" means here

Defined by policy, not by capability. See [ADR-0003](../decisions/0003-offline-scope.md).

| Surface | Offline? | Contents |
|---|---|---|
| **Operating surface** | Fully, reads and writes | Register, payment, refund, receipt reprint, cash movement, shift open/close, stock count, price override, customer lookup and create |
| **Administrative surface** | Online required, explicitly | Create/edit product, change price list, edit tax rules, user and permission management, store config, cross-store reporting, customer *record* edits and duplicate merges |

Customer capture is the one row that spans both surfaces, and the split is deliberate:
registering a walk-in customer at the till is an operational event (`CustomerRegistered`,
device-minted ID, insert-if-absent on ingest), while editing, merging or deleting a
customer record is master data and online-only. Two offline devices registering the same
person produce two records, resolved by a back-office merge rather than prevented. The
server-side model and its authorization, projection and reconciliation rules are in
[§2.4](02-server.md#two-write-models-and-knowing-which-you-are-in).

The line is drawn where it is because the two surfaces have opposite economics. Everything
a cashier touches during a trading day *must* work offline — that is the reason this
project exists. Back-office authoring is low-frequency, single-writer-in-practice, and
high-blast-radius: a tax rule merged wrong is a compliance problem, and allowing it to be
edited offline creates that risk to save a manager an inconvenience they will encounter
approximately never.

This is a policy line, not a technical wall. The sync engine could carry admin writes. If
the business later decides the conflict cost is worth it, the change is additive.

## 4.2 Replication tiers

See [ADR-0004](../decisions/0004-replication-tiers.md).

| Tier | Contents | Scope | Storage | Sync |
|---|---|---|---|---|
| **1 — Hydrated** | Catalog, prices, promos, tax rules, payment methods, customers, staff, permissions, store config, feature flags | `store_id` + tenant-global | IndexedDB → **in memory** | Full replica, delta pull |
| **2 — Resident** | Sales, payments, stock movements, shifts | `store_id`, rolling window (default 35 d) | IndexedDB, **queried by index** | Windowed pull, evictable |
| **3 — Outbox** | Unsynced local writes | This device | IndexedDB, **never evicted** | Push-only, drained on ack |
| **4 — Server-only** | Aggregates, audit log, cross-store analytics, history beyond the window | — | none | Online query |

Two distinctions that a naive "replicate the workspace" design misses:

**On-device ≠ in-memory.** Hydrating everything into an observable graph is correct for
Tier 1, which is read on every keystroke, and actively harmful for Tier 2, which is read
twice a day by point lookup. Conflating them is how a till tablet runs out of memory in
month eight.

**The outbox is its own tier with its own rules.** It is not an implementation detail of
the sync engine. It is the only data in the system that exists nowhere else, and it gets
durability guarantees nothing else gets.

### Sizing the window

The Tier-2 window is set by the **returns policy**, not by a storage guess. If returns are
accepted for 30 days, sales headers must be reachable on-device for 30 days plus a margin
— otherwise a return during an outage is impossible. Older returns fall back to an
online-only path, which is acceptable because they are rare and because a two-month-old
return during a network outage is a genuinely tolerable failure.

Note what the window is *not* sized by: the length of an outage. Trading through an outage
needs room in the *outbox* (forward writes), not depth of *history*.

## 4.3 The offline write path

Every operational write is an immutable domain event.

```
1. User completes sale
2. Client builds SaleCompleted:
     event_id     = UUID v7            (client-generated → idempotency key)
     device_id    = registered device
     device_seq   = monotonic, gapless (gap ⇒ data loss, alertable)
     occurred_at  = device clock       (recorded, untrusted)
     payload      = serialised protobuf
3. Durable append to outbox            ← AWAITED. UI does not proceed until this lands.
4. Apply projection to local store
5. UI renders, receipt prints
6. Sync engine drains when connectivity allows
```

**Step 3 must complete before step 5.** The optimistic-render-then-persist pattern is
correct for a todo list and wrong for a till. A crash in that window is cash taken with no
record — the exact failure this architecture exists to prevent.

### A sale cannot conflict

Two devices selling simultaneously do not conflict any more than two cash registers
conflict by both taking money. They produce two distinct facts. The server accepts both.
There is no merge function because there is nothing to merge.

This removes CRDTs and last-write-wins from roughly 95% of write volume — which is why the
event-sourced framing is the central decision rather than a stylistic one.

## 4.4 Conflict resolution

See [ADR-0005](../decisions/0005-event-sourced-writes.md) and
[ADR-0006](../decisions/0006-oversell-accepted.md).

| Data | Conflict possible? | Resolution |
|---|---|---|
| Sales, payments, refunds, cash movements | **No** | Distinct immutable facts. Both accepted. |
| Stock on hand | **No** — not a writable field | Projection of the movement log. Devices append movements; nobody writes a balance. |
| Shift totals | **No** | Projection of cash movements; tolerates late arrivals. |
| Master data (product, price, customer record) | Yes, rarely | LWW per field + version column. Mostly moot: online-only per §4.1, so it reduces to ordinary optimistic concurrency (409 + reload). |
| Customer registration at the till | **No** | Distinct immutable facts with device-minted IDs. Two devices registering one person yields two records — a back-office merge, not a conflict. |
| Receipt numbers | Structurally | Device-prefixed local sequence. See §4.6. |

### Stock: the trap

Stock on hand looks like a counter two devices decrement, so it looks like it needs a
counter CRDT. It does not. It needs to **not be a writable field at all**.

```
stock_on_hand(product, store) = Σ movements(product, store)
```

Devices append `StockMoved` events (sale, receipt, adjustment, transfer). The balance is
computed server-side. Offline oversell then stops being a data conflict and becomes a
*business event*: the log correctly records that you sold three of something you had two
of. The business handles it with a backorder, a substitution, or an apology.

Attempting to prevent this via distributed consensus is the wrong axis. **You cannot get
consensus during a partition** — that is what a partition is. The only way to guarantee no
oversell offline is to refuse to sell offline, which is worse for the business than
occasionally overselling. This is a business decision that needs explicit sign-off, not a
silent technical default. See [ADR-0006](../decisions/0006-oversell-accepted.md).

Local stock display is therefore *advisory*: last-known balance plus this device's
unsynced movements, labelled as approximate when the device has been offline beyond a
threshold. Showing a confident wrong number is worse than showing an honest uncertain one.

## 4.5 Sync protocol

```
      DEVICE                                    SERVER
        │                                          │
        │──── PushEvents(batch of ≤200) ──────────▶│
        │                                          │ per event:
        │                                          │  validate schema + authz
        │                                          │  INSERT ... ON CONFLICT DO NOTHING
        │                                          │  update projections (same txn)
        │◀─── PushEventsResponse(per-event) ───────│
        │     ACCEPTED | DUPLICATE                 │
        │     REJECTED | DEFERRED                  │
        │                                          │
        │ mark drained / quarantine / retry        │
        │                                          │
        │──── PullChanges(cursors per table) ─────▶│
        │◀─── stream ChangeSet ────────────────────│
        │     per ChangeSet, ONE IndexedDB txn:    │
        │       upsert rows + advance cursor       │
        │     cursor is durable only on commit     │
```

**Push before pull, always.** Local writes are the data that exists nowhere else; getting
them to durable storage takes precedence over freshness of the read cache.

### Rules

- **Idempotent by `event_id`.** A retry after a lost ack is a no-op. Given that lost acks
  are the normal condition on the networks this product targets, this is the single most
  important property in the protocol.
- **Ordered per device** by `device_seq`. Not globally ordered — global ordering across
  devices is neither achievable nor needed. Ingest does **not** enforce strictly in-order
  arrival: a batch resumed after a partition, or one event `DEFERRED` while its successors
  succeed, legitimately lands out of order. The server accepts each event on its own
  merits and records the resulting gap in `device_sequence_state`
  ([§2.4](02-server.md#event-storage)). A gap is a *temporary deferred gap* until it
  outlives the drain SLO, at which point it is a data-loss alert. Refusing out-of-order
  events instead would convert one stuck event into a permanently blocked device, which is
  the failure mode this protocol exists to avoid.
- **Cursors advance only on a committed local transaction.** Each streamed `ChangeSet` is
  applied client-side as one IndexedDB transaction covering every row upsert *and* that
  table's cursor update. A crash mid-stream then resumes from the last cursor that
  actually committed: rows may be re-delivered and re-upserted, which is harmless because
  upserts are idempotent, but no row is ever skipped. Advancing the cursor separately from
  the rows it covers is the one bug in this design that loses data silently and is
  invisible until a report is wrong.
- **Batched**, ≤ 200 events or ≤ 1 MB, whichever first. Resumable at batch granularity.
- **Partial success is normal.** One bad event never blocks the batch behind it.
- **Cursor-based pull**, per table, using a monotonic watermark. Not timestamps —
  timestamp cursors silently lose rows when transactions commit out of order across a
  clock boundary, and silent loss is the worst possible failure mode here.
- **Backoff**: exponential with full jitter, 1 s → 60 s. Full jitter matters because a
  store's twenty devices all reconnect at the same instant when the router comes back.
- **`DEFERRED`** covers genuine dependency ordering — a sale referencing a product created
  on another device that has not synced yet. Retried, not quarantined.
- **`REJECTED`** is a last resort, reserved for auth failure, schema violation, or writing
  to a store the device does not belong to. **Never** for a business-rule violation that
  already happened physically. A device offline for two days acted in good faith against a
  world it could not see; the server's job is to record reality, not to litigate it.

### Transport

`PushEvents` is unary — it needs a per-event response. `PullChanges` is server-streaming,
which matters for the first sync of a new device (a full catalog) and for a large delta
after a long outage: streaming gives progress feedback and bounded memory instead of one
enormous response.

WebSocket push for live invalidation is deliberately **not** in v1. Polling on an interval
(30 s foreground, longer background) plus a kick on reconnect is sufficient for POS —
nobody needs sub-second propagation of a price change — and it removes a stateful
connection-management problem from the critical path. Add it when a feature genuinely
requires live updates.

## 4.6 Receipt numbering

Underrated, and it bites everyone who defers it.

Receipts commonly need gapless per-store sequences for tax compliance. **A gapless global
sequence cannot be allocated offline.** Two devices offline cannot coordinate on "next
number" — the same impossibility as §4.4.

Resolution:

- **Device-local sequence with a device prefix** for the number staff and customers see:
  `ST01-TAB03-000142`. Gapless per device, allocated offline, printable immediately.
- **Server-assigned canonical fiscal sequence** at ingest time, if the jurisdiction demands
  one. Assigned in `received_at` order, gapless per store, appears on reprints and fiscal
  reports.

Retrofitting receipt numbering after go-live is painful and customer-visible. Settle it
before the first line of `sales` is written.

## 4.7 Shift reconciliation

Two devices on one shift, one goes offline, both post cash movements. Not a merge conflict
— a reconciliation report that must tolerate late arrivals.

Consequence: **shift close cannot be assumed final until every device on that shift has
drained its outbox.** The UI needs an explicit state — *"Shift closing — 1 device still
syncing (14 transactions)"* — and the shift report needs to be regenerable when late
events land, with the amendment visible rather than silently rewriting a number a manager
already counted cash against.

## 4.8 The duplicated domain logic problem

The largest ongoing tax in this architecture, stated plainly rather than buried.

To price a basket offline, the client must implement pricing, discounts, promotions and
tax. The server must implement them too, because the device is untrusted. **Two
implementations of the same rules in two languages.**

Not solvable by sharing code — Go and TypeScript do not share a runtime, and WASM-compiling
the Go pricing engine into the browser trades this problem for bundle size, debuggability,
and a much worse developer experience on the client's hottest path.

Contained by:

1. **Shared inputs.** Rule definitions are Protobuf messages, generated for both sides. The
   *data* never drifts; only the interpretation can.
2. **A shared golden-file corpus.** One JSON corpus of (basket, rules) → expected totals,
   in `api/testdata/pricing/`. Both the Go suite and the Vitest suite run against it. A
   divergence fails both CI jobs.
3. **Server recomputes on ingest and records both figures.** A mismatch is logged,
   alerted, and reported — it does not reject the sale, because the customer has already
   paid the amount the device displayed.

   Which figure is *authoritative* has to be stated, not left to whichever query is
   written first. Both are persisted on the sale projection:

   | Field | Meaning | Used by |
   |---|---|---|
   | `charged_total` | What the device displayed and the customer actually paid | **Authoritative.** Sale projection, receipt, refund ceiling, till and shift reconciliation, tax reporting, accounting export |
   | `recomputed_total` | What the server's pricing engine says the basket should have cost | Diagnostics only: mismatch metric, variance reporting, and the manager task raised on a mismatch |

   The rule follows from the same principle as §4.4: the physical world wins. Money
   changed hands at `charged_total`, so that is the figure the books must balance to and
   the ceiling a refund may not exceed. Reporting on `recomputed_total` would produce a
   ledger that disagrees with the cash in the drawer, which is the failure mode this whole
   architecture is built to avoid. `recomputed_total` never silently rewrites anything; a
   variance is a *reported* number carrying a `variance_amount` and, above a configurable
   threshold, a manager task. Correcting one is an explicit compensating event
   (an adjustment or refund), never an edit.
4. **A mismatch rate metric.** Non-zero divergence is a bug with a dollar value attached,
   and it is visible on a dashboard rather than discovered at month-end.

Corollary: keep the pricing rule language **small and declarative**. Every expressive
feature added to promotions is implemented twice and tested twice. This is a strong,
ongoing argument against a Turing-complete promotion engine.

## 4.9 Bootstrapping a new device

1. Register device online — obtains device ID, store binding, credentials.
2. Full Tier-1 pull (streamed, with progress). Largest single transfer the device makes.
3. Tier-2 window pull, newest first, so the returns-capable window is usable before the
   pull finishes.
4. Service worker precaches route chunks.
5. Device marked till-ready.

Steps 1–3 require connectivity. A device cannot be provisioned offline, which is correct —
the alternative is sideloading a trust anchor, which is a worse problem.

For a large catalog, Tier-1 bootstrap should be served as a **signed snapshot from MinIO**
rather than streamed row-by-row from Postgres: a single compressed object is
dramatically faster over a store's poor connection and is CDN-cacheable, with deltas
applied on top from the snapshot's watermark.

## 4.10 Observability

Sync failures are silent by nature. A store can be offline for a day and nobody notices
until the numbers are wrong. Instrument accordingly:

| Signal | Alert |
|---|---|
| Oldest undrained outbox entry age, per device | > 1 h during trading hours |
| Outbox depth, per device | > 500 |
| Quarantined event count | > 0 |
| Device last-seen | > 4 h during trading hours |
| `device_seq` gap open in `device_sequence_state` | still open after the drain SLO — indicates data loss. A gap inside the SLO is normal out-of-order arrival |
| `event_id` or `device_seq` reuse rejection | any — indicates a client bug or a cloned device |
| Pricing mismatch rate | > 0 |
| Oversell events | reported daily, not alerted |
| Pull cursor lag | > 15 min |

Per-device, not aggregate. An aggregate sync-health number hides the one store that has
been dark since Tuesday.

---

## Tradeoffs

**Event sourcing costs conceptual load.** Two write models, and every developer must know
which one they are in. Reporting reads projections, not the log. The alternative —
LWW-syncing a mutable `sales` row — is simpler right up until the first unreconcilable
till, at which point it is unrecoverable because the history needed to reconstruct the
truth was never stored. Accepted deliberately.

**Accepting oversell is a business decision with real cost.** Backorders, customer
disappointment, staff explaining a stockout. The alternative is refusing to sell during an
outage, which is a larger and more certain cost. Needs explicit business sign-off, and the
mitigations (soft warnings, per-category thresholds, prompt reconciliation) should be
funded rather than assumed.

**Advisory stock numbers will generate support tickets.** "It said 4 in stock." Contained
by labelling staleness honestly and by keeping the sync window tight, not eliminated.

**35 days of resident history is a guess** until the returns policy is known. If returns
run to 90 days on a high-volume store, the IndexedDB budget needs re-checking against real
hardware — this is a measurement, not an estimate.

**No live push in v1** means a price change can take up to 30 s to reach a till. Fine for
retail. Not fine if a future feature needs live queue management or real-time
cross-device basket handoff. Adding WebSocket later is additive; the polling design does
not have to be undone.

**Duplicated pricing logic** — §4.8. The mitigations contain it; they do not remove it.
Budget for the golden corpus as real, ongoing work.

**Protobuf payloads are opaque to SQL forensics.** Mitigated by projections and a decode
CLI. Genuinely painful during an incident, which is why the CLI should exist before the
first incident.

## Open Questions

1. **Returns window** — sets the Tier-2 window. Blocks storage budgeting.
2. **Worst-case supported offline duration** — sets outbox sizing and eviction policy.
3. **Fiscal requirements per jurisdiction.** Mandated real-time reporting or certified
   fiscal devices would materially constrain, and in the limit invalidate, this design.
   Highest-severity unknown.
4. **Card payments offline.** Cloud-only terminals cannot authorise offline. If most
   transactions are card, "offline sale" may mean "cash only" — which changes what this
   architecture is worth. Highest-value unknown.
5. **Manager reconciliation UX** for quarantined events. Needs design before build; it is
   the human end of the whole protocol and is easy to leave until it is urgent.
6. **Do devices ever need peer-to-peer sync** (store LAN up, internet down)? Would let one
   device act as a local relay and materially improve multi-device stores during ISP
   outages. Significant added complexity. Explicitly out of scope for v1; note it as the
   most likely v2 architectural change.
