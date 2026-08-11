# 1. High-Level Architecture Overview

> Status: proposed · Date: 2026-08-11 · Supersedes: nothing
> Decision records: [`docs/decisions/`](../decisions/)

## 1.1 What this system is

A multi-tenant, multi-store point-of-sale platform. A tenant is a retail business; a
tenant has one or more stores; a store has one or more devices; a device is a browser
running an installed PWA.

The defining constraint is not scale and it is not latency. It is this:

> **A store must be able to take money while the network is gone, for hours, and every
> cent taken must still be reconcilable afterwards.**

Everything below follows from that sentence. Where a design choice trades convergence
convenience against auditability of money, auditability wins.

## 1.2 The framing that drives the design

An earlier draft of this architecture reasoned from Linear's local-first model
("replicate the workspace into the browser, converge, treat the server as a sync
target"). That model solves *perceived latency* for a collaborative document product.
It is the wrong center of gravity here.

| | Collaborative doc app | POS |
|---|---|---|
| Problem solved | Network exists, feels slow | Network is genuinely gone |
| Cost of a lost write | Retype it | Unreconcilable till, tax exposure |
| Cost of a stale read | Mild confusion | Sold at the wrong price |
| Conflict semantics | Any convergent outcome is fine | Accounting defines the correct outcome |
| Data shape | Bounded workspace, slow-growing | Unbounded append-only financial stream |
| Offline duration | Seconds to minutes | Hours to days |
| Concurrent writers per entity | Constant — that *is* the product | Rare on master data, never on a sale |

So the architecture is stated as:

> **POS is an offline event-sourcing problem with a cached read model on the side.**

Not a replicated-document problem. Postgres remains the ledger of record. The server
*validates*, it does not merely merge. The client holds a scoped read cache plus a
durable outbox — it is not a peer replica.

What *does* transfer from the local-first playbook is the client-side tactics: optimistic
local writes with synchronous re-render, granular reactivity, local-only search,
aggressive code splitting, service-worker precache, composited-only animation. On a POS
these are not polish. Scan-to-scan time at a busy till is a throughput number, and a
200 ms hitch per line item is queue time a store manager can measure.

See [ADR-0003](../decisions/0003-offline-scope.md) and
[ADR-0005](../decisions/0005-event-sourced-writes.md).

## 1.3 System boundaries

```
┌─────────────────────────────────────────────────────────────────────┐
│ DEVICE (installed PWA — till tablet, phone, back-office desktop)    │
│                                                                     │
│  React + TanStack Start                                             │
│    ├── Read path  ── TanStack Query ──▶ local store (IndexedDB)     │
│    ├── Write path ── domain event ────▶ OUTBOX (durable, ordered)   │
│    └── Sync engine ── push outbox / pull deltas ──┐                 │
└───────────────────────────────────────────────────┼─────────────────┘
                                                    │ HTTPS
                                                    │ (gRPC-Web / REST)
┌───────────────────────────────────────────────────▼─────────────────┐
│ EDGE — gateway/ingress: TLS, routing, rate limit, OIDC token check  │
└───────────────────────────────────────────────────┬─────────────────┘
                                                    │
┌───────────────────────────────────────────────────▼─────────────────┐
│ SERVER — Go modular monolith (go-kratos), one deployable            │
│                                                                     │
│  transport: gRPC + REST (generated from the same .proto)            │
│  ┌────────┬──────────┬───────────┬────────┬──────────┬───────────┐  │
│  │identity│ catalog  │ pricing   │ sales  │inventory │ reporting │  │
│  ├────────┴──────────┼───────────┴────────┴──────────┴───────────┤  │
│  │ shifts · customers│ (nine modules total — see §2.2)           │  │
│  └───────────────────┴───────────────────────────────────────────┘  │
│  ┌───────────────────────── sync module ───────────────────────────┐│
│  │ ingest(events) · idempotency · delta(cursor) · replication policy││
│  └─────────────────────────────────────────────────────────────────┘│
└───────┬──────────────┬───────────────┬────────────────┬─────────────┘
        │              │               │                │
   ┌────▼────┐   ┌─────▼────┐   ┌──────▼─────┐   ┌──────▼──────┐
   │Postgres │   │  Redis   │   │   MinIO    │   │  Unleash    │
   │ ledger  │   │  cache   │   │  objects   │   │  flags      │
   │+ FTS    │   │ + locks  │   │  receipts  │   │             │
   └─────────┘   └──────────┘   └────────────┘   └─────────────┘
```

Three boundaries matter, and each is a place where a contract is enforced rather than
assumed:

1. **Device ↔ Edge.** Untrusted. Every payload is validated against the Protobuf schema
   *and* against business invariants. A device is a hostile input source — it runs on
   hardware staff can physically access and whose clock they can change.
2. **Module ↔ Module** inside the monolith. In-process Go interface calls, no network.
   Enforced by package visibility and an import linter, not by convention.
3. **Server ↔ Postgres.** The only writer. There is no path from a device to the
   database that does not pass through server-side validation.

## 1.4 How the pieces fit

**Frontend and mobile are one codebase.** Mobile is the same React app installed as a
PWA. There is no native codebase, no React Native, no separate design system. A till
tablet, a manager's phone, and a back-office desktop run identical bundles with
different route access and different replication scope.
See [ADR-0010](../decisions/0010-pwa-not-native.md) — including its honest limits, which
are real and may eventually force a thin native shell.

**The server is a modular monolith.** One binary, one deploy, one database, hard internal
module boundaries designed so that a module *could* be extracted later without rewriting
its callers. It is not microservices, and it is not a ball of mud that hopes to become
microservices. See [ADR-0001](../decisions/0001-modular-monolith.md).

**Protobuf is the single source of truth for every contract**, including the offline
event envelope. Go types and TypeScript types are both generated from it; neither is
hand-written; drift between them is a build failure, not a bug report.
See [ADR-0002](../decisions/0002-protobuf-buf-contracts.md).

**Local-first is scoped by tier, not applied uniformly.** Four tiers with different
durability, storage, and sync rules. See [ADR-0004](../decisions/0004-replication-tiers.md)
and [§4 Local-first sync strategy](04-sync.md).

## 1.5 Monorepo layout

```
.
├── api/                        # ── Protobuf: THE contract source of truth
│   └── pos/
│       ├── common/v1/          #    money, ids, pagination, audit envelope
│       ├── identity/v1/
│       ├── catalog/v1/
│       ├── pricing/v1/
│       ├── inventory/v1/
│       ├── sales/v1/
│       ├── shifts/v1/           #    ShiftOpened, CashMoved
│       ├── customers/v1/        #    CustomerChanged
│       ├── reporting/v1/
│       └── sync/v1/            #    event envelope, push/pull, cursors
│
├── apps/
│   ├── server/                 # ── Go modular monolith (go-kratos)
│   │   ├── cmd/server/
│   │   ├── internal/
│   │   │   ├── conf/
│   │   │   ├── server/         #    grpc + http wiring, middleware chain
│   │   │   ├── module/         #    ONE dir per bounded context
│   │   │   │   ├── catalog/{service,biz,data}/
│   │   │   │   ├── sales/{service,biz,data}/
│   │   │   │   └── ...
│   │   │   └── platform/       #    cross-cutting: db, redis, minio, otel, flags
│   │   └── migrations/
│   │
│   └── web/                    # ── React + TanStack Start PWA
│       ├── app/routes/
│       ├── src/{features,components,lib}/
│       └── tests/{unit,e2e}/
│
├── gen/                        # ── buf generate output. Committed. Never edited.
│   ├── go/
│   └── ts/
│
├── packages/                   # ── shared TypeScript
│   ├── contracts/              #    re-export gen/ts + proto→Zod bridge
│   ├── sync-engine/            #    outbox, cursor, reconciliation (framework-free)
│   ├── local-db/               #    IndexedDB schema + migrations
│   ├── ui/                     #    shadcn/ui primitives + POS components
│   └── config/                 #    eslint / ts / tailwind / vitest presets
│
├── deploy/
│   ├── tofu/{modules,envs}/    #    OpenTofu IaC
│   ├── compose/                #    local dev (Docker) + prod quadlets (Podman)
│   └── observability/          #    Prometheus, Grafana, Loki, Jaeger config
│
├── docs/
│   ├── architecture/           #    this directory
│   ├── decisions/              #    ADRs
│   ├── runbooks/               #    deploy, rollback, DR, incident
│   └── templates/              #    README / ADR / runbook templates
│
├── scripts/
├── buf.yaml  buf.gen.yaml  buf.lock
├── go.work                     #    apps/server + gen/go
├── pnpm-workspace.yaml
└── lefthook.yml
```

### Why this shape

**`api/` is a sibling of `apps/`, not a child of either.** Neither Go nor TypeScript owns
the contract. Putting protos inside `apps/server` would make the server the de-facto
owner and invite server-shaped API design — request/response types that mirror the Go
domain model rather than what a client actually needs.

**`gen/` is committed.** Contested, so the reasoning: a fresh clone builds without a
protoc toolchain, IDE navigation works immediately, and code review shows the actual
contract diff — a reviewer sees that a field became `optional` rather than inferring it
from a `.proto` line. The cost is merge conflicts in generated files and the ever-present
risk of a hand-edit. Both are handled by a CI job that regenerates and fails on any diff,
plus a `CODEOWNERS`/`.gitattributes` marking `gen/` as generated.

**`packages/sync-engine` has no React dependency.** The hardest, most correctness-critical
logic in the client is the outbox and reconciliation. Keeping it framework-free means it
is testable in plain Vitest with a fake IndexedDB and a scripted network, with no
component rendering involved. This is the one package that would justify a property-based
test suite.

## 1.6 Data flow: the two paths

**Read path (every screen, always local first).**

```
Component → TanStack Query (queryFn reads local-db, never fetch)
          → IndexedDB / in-memory store
          → render synchronously
                      ⋮
          sync engine pulls deltas in background → writes local-db
          → query cache invalidated → surgical re-render
```

There is no loading spinner on a Tier-1 read. Ever. If the data is not local, that is a
sync bug or a deliberate Tier-4 (server-only) screen, and Tier-4 screens say so in the UI.

**Write path (operational writes).**

```
User action
  → build domain event (client UUID, device id, device seq, device clock)
  → append to OUTBOX  ← durable write, awaited
  → apply projection to local store
  → UI updates                                  [all of the above: no network]
                      ⋮
  → sync engine drains outbox → server validates → Postgres (event + projection,
                                                   one transaction)
  → ack by event UUID → outbox entry marked drained
```

The ordering is deliberate and non-negotiable: **the outbox write is awaited before the
UI acknowledges the sale.** Optimistic-render-then-persist is correct for a todo app and
wrong here — a crash in that window is a sale that took cash and left no record. The
durable append is on the order of a millisecond; it is not the thing making the till slow.

## 1.7 Environments

| | Local dev | CI | Staging | Production |
|---|---|---|---|---|
| Containers | Docker Compose | GitHub Actions runners | Podman quadlets | Podman quadlets |
| Postgres | container | Testcontainers | managed / VM | managed, PITR + replica |
| Search | Postgres FTS | Postgres FTS | Postgres FTS | Postgres FTS |
| Secrets | `.env.example` | Actions secrets | OpenBao | OpenBao |
| IaC | n/a | plan only | OpenTofu apply | OpenTofu apply, gated |

The Docker-local / Podman-prod split is a real inconsistency, deliberately accepted with
a mitigation. See [ADR-0011](../decisions/0011-podman-prod-docker-local.md).

---

## Tradeoffs

**Monorepo over polyrepo.** One atomic commit changes a proto, the Go handler, and the
React caller together — which is the single biggest reason to choose it, given that
contract drift between Go and TypeScript is the failure mode this project is most exposed
to. The cost is CI time and tooling complexity: every push risks running the full matrix.
Mitigated with path-filtered workflows and Buf/Turbo caching, but expect to spend real
effort on CI performance by the time the repo has three developers in it.

**One repo, two language ecosystems.** `go.work` and `pnpm-workspace.yaml` coexist and
neither knows about the other. Task orchestration across them is glue you own. Nx/Bazel
would unify it and cost far more than it returns at this size — revisit only if the
number of deployable artifacts exceeds roughly five.

**Committed generated code.** Stated above. The failure mode to watch is somebody
hand-editing `gen/` under deadline pressure and CI's regenerate-and-diff job being the
only thing that catches it. Make that job fast and un-skippable.

**"Offline-complete for the operating surface" is a policy line, not a technical one.**
The sync engine is capable of carrying admin writes too. Drawing the line where we drew
it is a deliberate reduction of conflict surface. Expect to be asked "why can't I create
a product offline?" and expect that the answer is a business conversation rather than a
technical one.

**The device is treated as hostile.** All pricing, tax, discount and permission decisions
are *re-validated server-side* on ingest even though the client already computed them.
This means the pricing rules exist in two places — Go and TypeScript — generated from
shared Protobuf but implemented twice. That duplication is the price of both offline
operation and trustworthy numbers. It is the largest ongoing tax in this architecture and
[§4.8](04-sync.md#48-the-duplicated-domain-logic-problem) addresses how to contain it.

## Open Questions

1. **Tenancy isolation model.** Shared schema with `tenant_id` on every table, versus
   schema-per-tenant, versus database-per-tenant. Affects migrations, backup granularity,
   noisy-neighbour blast radius, and the credibility of the answer we give to an
   enterprise security questionnaire. Needs deciding before the first migration is
   written. *Provisional lean: shared schema + `tenant_id` + Postgres RLS as a
   defence-in-depth backstop.*
2. **Expected scale.** Stores per tenant, devices per store, transactions per device per
   day, and the p99 basket size. These numbers change the replication window, the
   IndexedDB budget, and whether Postgres FTS holds. Currently unknown.
3. **Fiscal/regulatory jurisdiction.** Some jurisdictions mandate certified fiscal
   devices, gapless signed sequences, or government real-time reporting. Any one of these
   materially constrains the offline design — real-time fiscal reporting is close to
   incompatible with taking payment offline. Must be answered before build starts.
4. **Payment terminal integration model.** Cloud-only card terminals cannot authorise
   offline at all, which means "offline sale" may mean "cash and stored-value only" in
   practice. This determines what the offline experience actually *is* for a customer
   holding a card, and it is the question most likely to change the product.
5. **Worst-case offline duration** the business will commit to supporting. Drives outbox
   sizing, IndexedDB quota strategy, and whether browser storage eviction forces a native
   shell.
