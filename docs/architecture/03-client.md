# 3. Client — React + TanStack Start PWA

## 3.1 The one rule

> **No component ever calls the network. Components read the local store. The sync engine
> owns the network.**

Every other client decision follows. If a component can `fetch`, then some screen
eventually works only online, and the offline guarantee becomes "works offline except the
parts nobody tested offline." Enforced by an ESLint rule banning `fetch`/transport imports
outside `packages/sync-engine`.

## 3.2 Structure

```
apps/web/
├── app/
│   ├── routes/
│   │   ├── _till/              # the money path — its own layout, its own budget
│   │   │   ├── register.tsx
│   │   │   ├── payment.tsx
│   │   │   └── receipt.$id.tsx
│   │   ├── _ops/               # shift, stock count, returns — offline-capable
│   │   ├── _admin/             # ONLINE-ONLY (ADR-0003). Guarded by one boundary.
│   │   └── _auth/
│   ├── router.tsx
│   └── root.tsx
├── src/
│   ├── features/               # vertical slices, mirroring server modules
│   │   ├── sales/{components,hooks,domain,queries.ts,commands.ts}
│   │   ├── catalog/
│   │   ├── inventory/
│   │   └── shifts/
│   ├── components/             # app-wide shells, no feature knowledge
│   └── lib/
└── tests/{unit,e2e}/

packages/
├── sync-engine/                # outbox, cursors, reconciliation. NO React.
├── local-db/                   # IndexedDB schema + migrations
├── contracts/                  # gen/ts re-export + proto→Zod bridge
└── ui/                         # shadcn/ui primitives + POS components
```

Features are vertical slices named after server modules. A cross-cutting `components/`,
`hooks/`, `types/` layout produces the situation where changing one behaviour touches six
directories. Slices keep a change local, and the naming symmetry with the server means
"where does this live" has the same answer on both sides.

`queries.ts` (reads) and `commands.ts` (writes) are separate files per feature because
they obey different rules — reads are local and synchronous, writes go through the
outbox. Separating them makes the asymmetry visible instead of something you learn by
being burned.

## 3.3 TanStack Start: what we use and what we don't

TanStack Start is a full-stack framework with SSR and server functions. Most of that is
inapplicable to a POS: SSR needs a server, and a till with no network has no server.

| Feature | Used | Why |
|---|---|---|
| File-based routing | ✅ | Type-safe routes, good code splitting |
| Type-safe params/search | ✅ | Search params are real state (filters, date ranges) |
| Route-level code splitting | ✅ | Till bundle must not carry admin/reporting code |
| Loaders | ⚠️ limited | Only to trigger local hydration, never to fetch |
| SSR | ❌ | Client-rendered SPA. Ships as a static bundle. |
| Server functions | ⚠️ admin-only | Acceptable on online-only admin routes |

We are using it as a very good typed client-side router. That is a legitimate use, but it
is worth naming: if the router is all we need, TanStack Router alone is the smaller
dependency. Start is chosen for the migration path — an eventual marketing site,
server-rendered receipt pages, or an admin surface that benefits from SSR — and that path
should be real within a year or the dependency should be reconsidered.

The `_admin` route group is a **hard boundary**, not a convention. Its layout route checks
connectivity and renders an explicit offline state rather than a broken form. This is what
turns ADR-0003's policy line into something a user experiences as designed rather than
broken.

## 3.4 Data layer

Three layers, each with a single responsibility:

```
┌──────────────────────────────────────────────────────────┐
│ Components                                               │
│   useQuery / useMutation — never aware of the network    │
├──────────────────────────────────────────────────────────┤
│ TanStack Query                                           │
│   queryFn reads local-db. Cache + invalidation only.     │
├──────────────────────────────────────────────────────────┤
│ local-db (IndexedDB, via Dexie)                          │
│   Tier 1 mirrored to memory; Tier 2 queried by index     │
├──────────────────────────────────────────────────────────┤
│ sync-engine                                              │
│   OUTBOX (push) · cursors (pull) · reconciliation        │
└──────────────────────────────────────────────────────────┘
```

**TanStack Query is used as a reactive cache over local storage, not as a data-fetching
library.** Its `queryFn` reads IndexedDB; it never touches the network. We keep it for
what it is genuinely good at — dependency-tracked invalidation, `useSuspenseQuery`,
devtools — while `staleTime: Infinity` and `gcTime: Infinity` disable everything that
assumes a server.

This is worth a moment of honesty: it is an unconventional use, and a new team member will
assume `queryFn` fetches. Mitigation is a documented custom `useLocalQuery` wrapper that
makes the intent explicit at the call site, plus the ESLint ban on `fetch`.

```ts
// features/catalog/queries.ts
export const productSearch = (storeId: string, term: string) =>
  queryOptions({
    queryKey: ['catalog', 'search', storeId, term],
    queryFn: () => localDb.searchProducts(storeId, term),   // IndexedDB. No network.
    staleTime: Infinity,
    gcTime: Infinity,
  });
```

### Tier 1 in memory

Catalog, prices, promos, tax rules, staff, permissions and store config are hydrated from
IndexedDB into an in-memory index at boot. They are read on every keystroke of a product
search and every line-item price calculation; an IndexedDB round-trip per keystroke is
measurable in a way that shows up as scan-to-scan time.

Tier 2 (historical sales) is **never** hydrated wholesale. It is queried by index for the
two workflows that need it — reprint a receipt, return against a sale — both of which are
point lookups. Hydrating it would be the single fastest way to blow the memory budget on
a cheap tablet.

### The outbox

The most important structure in the client. Everything else can be rebuilt from the
server; the outbox is the only place holding data that exists nowhere else.

```ts
interface OutboxEntry {
  eventId: string;        // UUID v7 — sorts by creation time
  deviceSeq: number;      // monotonic, gapless, survives restart
  type: string;           // fully-qualified proto message name
  payload: Uint8Array;    // serialised protobuf — same bytes the server will validate
  createdAt: number;      // device clock, recorded not trusted
  attempts: number;
  status: 'pending' | 'inflight' | 'drained' | 'quarantined';
  lastError?: string;
}
```

Rules, each of which exists because violating it loses money:

1. **Durable append is awaited before the UI acknowledges the sale.** Not optimistic. A
   crash between "receipt printed" and "event persisted" is a sale that took cash and left
   no record.
2. **`deviceSeq` is gapless and monotonic across restarts.** A gap is how the server
   detects loss. Allocated inside the same IndexedDB transaction as the append.
3. **Entries are never deleted until the server acks by `eventId`.** Not on "probably
   sent", not on optimistic assumption.
4. **`quarantined` entries surface as a manager task.** Never silently dropped.
5. **The unsynced count is always visible in the UI.** Staff need to know that closing the
   store with 340 unsynced transactions is a thing to mention to someone.
6. **Payload is serialised at enqueue time**, using the same Protobuf types the server
   validates. A payload built at drain time can be built from a mutated local store and
   no longer represent what the cashier actually did.

Storage eviction is the sharpest risk in the entire client design. Browsers evict
IndexedDB under pressure, and losing the outbox means losing sales. Mitigations:
`navigator.storage.persist()` requested at install; installed PWAs are granted persistent
storage on all target platforms; quota is monitored and surfaced before it is critical;
Tier 2 history is evicted first and aggressively; the outbox is never evicted by our own
code. Residual risk is real and is the strongest argument for a native shell —
see [ADR-0010](../decisions/0010-pwa-not-native.md).

## 3.5 Write path

```ts
// features/sales/commands.ts
export async function completeSale(draft: SaleDraft) {
  const event = buildSaleCompleted(draft);      // pure; priced from local Tier 1 rules

  await outbox.append(event);                   // durable. awaited. non-negotiable.
  await localDb.applyProjection(event);         // local read model updated
  queryClient.invalidateQueries({ queryKey: ['sales'] });

  syncEngine.kick();                            // fire-and-forget; may be offline
}
```

No spinner, no rollback path, no error boundary around the network — because there is no
network in this function. The only failure mode is storage failure, which is fatal and
must block the sale loudly rather than degrade quietly. A POS that silently fails to
record a sale is worse than a POS that refuses to sell.

**Server rejection does not roll back the local state.** The sale happened; cash changed
hands. A rejected event becomes a reconciliation task for a manager, not a disappearing
row. This differs from standard optimistic-update libraries, which assume the server is
the arbiter of whether an action occurred. Here the physical world is.

## 3.6 Forms and validation

React Hook Form + Zod. The rule that matters: **Zod schemas are derived from Protobuf, not
hand-written.** Hand-written client validation drifts from server validation, and the
symptom is a form that passes locally and is rejected after a two-day sync — the worst
possible time to discover a validation mismatch.

`packages/contracts` generates base Zod schemas from the proto (structural: types,
required, ranges from `protovalidate` rules). Features extend them with UI-only concerns
(confirm-password, cross-field warnings). Structural rules are never re-declared by hand.

## 3.7 Performance

These are throughput requirements, not polish. Scan-to-scan time is a number a store
manager can measure.

| Target | Budget | Why |
|---|---|---|
| Cold boot to interactive till | < 2 s | Morning open, cheap tablet |
| Warm boot (service worker) | < 500 ms | Tab reload mid-shift |
| Add line item to basket | < 16 ms | One frame. Faster than a barcode scanner can re-arm. |
| Product search keystroke → results | < 50 ms | Feels instant |
| Complete sale → receipt | < 100 ms | Includes durable outbox write |

Techniques, adopted from the local-first playbook because they are correct here:

- **Route-level splitting** with the till path as its own chunk, preloaded first. Admin,
  reporting and charting code never enter the till bundle.
- **Per-package vendor chunks** so a dependency bump invalidates one chunk, not the whole
  vendor bundle. Matters when devices update over a store's poor connection.
- **`modulepreload` for critical chunks in `<head>`** to collapse the fetch→parse→fetch
  waterfall into one parallel batch.
- **Service worker precache** of all route chunks after first load. Combined with
  IndexedDB this is what makes a cold start work with no network at all.
- **Inline critical CSS** for the boot shell so the first paint is themed and correctly
  laid out before the first bundle arrives.
- **Granular reactivity.** Adding a line item re-renders one row, not the basket. Selector
  hooks + `memo` at row level; verified by a React Profiler assertion in the e2e suite
  rather than trusted.
- **Composited-only animation** (`transform`, `opacity`). Never `width`/`height`/`margin`.
  Durations 100–250 ms; instant appear, 150 ms dismiss.
- **`target: 'esnext'`**, no legacy polyfills. Device browser baseline is a procurement
  decision, stated in the README.

## 3.8 Keyboard and input

A till is a keyboard-first, and often keyboard-*only*, environment. Design accordingly:

- Barcode scanners emulate a keyboard. The register route needs a scanner-aware global
  handler that distinguishes machine-speed input from human typing (inter-keystroke
  timing) and routes it without requiring focus in a specific field.
- Every till action has a single-key or modifier shortcut, and shortcuts are visible in
  the UI rather than documented in a PDF.
- A command palette (`⌘K` / `Ctrl+K`) searches products, customers, actions and settings
  against the local Tier-1 index. No network, no debounce-for-the-server, instant.
- Touch targets sized for a fingertip on a greasy screen, not a mouse cursor. Full
  keyboard operation without a mouse is a hard requirement, and so is full touch operation
  without a keyboard — both device profiles exist in the same store.

## 3.9 Testing

| Layer | Tool | Focus |
|---|---|---|
| Domain logic (pricing, tax, totals, change due) | Vitest, pure | Exhaustive. This is money. Property tests for rounding. |
| `sync-engine` | Vitest + fake-indexeddb + scripted network | Partition, resume, duplicate, reorder, quarantine |
| Components | Vitest + Testing Library | Behaviour, not snapshots |
| Routes / flows | Playwright | Full till flows |
| Offline | Playwright + CDP offline + storage inspection | See below |

The offline e2e suite is what actually validates the architecture:

- Take five sales offline, close and reopen the tab, verify all five survive and drain
- Kill the browser mid-sale, verify no partial sale and no lost sale
- Two devices (two browser contexts) sell the last unit, verify both events land and
  oversell is reported
- Come back online after a simulated 48 h with a large backlog; verify ordering, progress
  feedback, and that the till stays usable while draining
- Quarantine flow: force a `REJECTED` event and verify it reaches the manager task list

Coverage target is 100% per project standards, with the caveat that coverage on the
domain and sync-engine packages is meaningful while coverage on presentational components
mostly measures how many components were rendered once.

---

## Tradeoffs

**TanStack Query as a local cache** is off-label. We get excellent invalidation and
devtools; we get a library whose entire documentation assumes a server, which will mislead
new contributors. The alternative — a purpose-built observable store (Zustand/MobX/Legend
State) — is more honest about what it is and less familiar. We chose Query because it is
already in the stack and the wrapper makes the intent explicit. **Revisit if** the wrapper
grows past roughly 100 lines of fighting the library's assumptions.

**IndexedDB via Dexie.** A dependency for something the platform provides. Raw IndexedDB
is a genuinely hostile API and the outbox is the last place to hand-roll transaction
handling. Worth the dependency.

**Ban on `fetch` in components** is absolute and will occasionally be annoying — an admin
screen that only ever runs online must still route through the sync engine's transport.
Accepted: the alternative is an offline guarantee that erodes one convenient exception at
a time.

**PWA storage eviction** is an accepted, monitored risk, not a solved problem. It is the
most likely reason this architecture changes.

**Two implementations of pricing and tax** (Go and TypeScript) is the largest ongoing tax.
Shared Protobuf inputs and a shared golden-file test corpus contain it; they do not
eliminate it. See [§4.8](04-sync.md#48-the-duplicated-domain-logic-problem).

**Client-side and server-side search will rank differently** unless deliberately held in
sync. Shared corpus, shared golden files, run in both CI jobs.

## Open Questions

1. **Device browser baseline.** Determines `esnext` targeting, `storage.persist()`
   behaviour, and whether an old Android WebView must be supported. This is a procurement
   question with direct architectural consequences.
2. **Receipt printing.** Web Serial / Web USB / Web Bluetooth support varies by platform,
   and is absent on iOS Safari. If thermal printers are required on iOS, the PWA-only
   decision fails and a native shell becomes mandatory. Must be answered before
   [ADR-0010](../decisions/0010-pwa-not-native.md) can move from proposed to accepted.
3. **Cash drawer and payment terminal access** — same question, same consequences.
4. **Multi-account on one device.** Shift handover between cashiers on a shared tablet:
   does each user get isolated local state, or is the device the identity with users as
   attribution on events? Second option is far simpler; needs confirming against how
   stores actually operate.
5. **Client update strategy.** Forcing a reload mid-shift is unacceptable; running a
   month-old client is a support problem. Likely answer is version-gated at shift
   boundaries, but the interaction with an outbox holding events serialised by the *old*
   schema needs design.
