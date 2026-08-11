# 2. Server — Go Modular Monolith (go-kratos)

## 2.1 Shape

One binary. One database. One deployment unit. Internal boundaries enforced by tooling,
not by good intentions.

The point of the modular monolith here is *not* "microservices later." It is that this
system has one transactional heart — a sale must atomically write an event, update
inventory movements, and update a shift total — and distributing that across services
turns a Postgres transaction into a saga. Sagas are how you get an unreconcilable till.
Keep money in one transaction.

See [ADR-0001](../decisions/0001-modular-monolith.md).

## 2.2 Module boundaries

A module is a bounded context. It owns its tables, exposes a Go interface, and has one
reason to change.

| Module | Owns | Emits | Notes |
|---|---|---|---|
| `identity` | tenants, stores, users, roles, devices, sessions | `DeviceRegistered` | Only module that mints/validates auth |
| `catalog` | products, variants, categories, barcodes, units | `ProductChanged` | Master data, LWW, mostly online-write |
| `pricing` | price lists, promotions, tax rules, discounts | `PriceChanged` | Pure calculation + rule storage |
| `inventory` | stock movements, counts, transfers, suppliers | `StockMoved` | Balances are **projections**, never written |
| `sales` | orders, line items, tenders, refunds, receipts | `SaleCompleted`, `RefundIssued` | The append-only heart |
| `shifts` | shift open/close, cash movements, till reconciliation | `ShiftOpened`, `CashMoved` | Tolerates late-arriving events |
| `customers` | customers, loyalty, contact details | `CustomerChanged` | Master data + PII boundary |
| `reporting` | read-only projections and aggregates | — | Reads other modules' projections; writes nothing |
| `sync` | event ingest, idempotency, deltas, cursors, replication policy | — | The device-facing contract |

### Rules that make the boundaries real

1. **No cross-module table access.** `sales` may not `SELECT` from `catalog` tables. It
   calls `catalog.Reader`. Enforced by `depguard` in `golangci-lint`, keyed on package
   path, so a violation fails CI rather than review.
2. **Modules depend on interfaces defined in the *consumer*, not the producer.** `sales`
   declares the narrow `ProductLookup` interface it needs; `catalog` satisfies it. This
   is interface segregation applied at module scale, and it means `sales` tests need a
   three-method fake rather than the whole catalog.
3. **Cross-module calls are synchronous in-process.** No internal message bus. An event
   bus between modules in a single process adds asynchrony and debugging pain to buy
   decoupling we can get more cheaply from interfaces. The *domain* events in §2.5 are a
   persistence concept, not an internal transport.
4. **One module may not import another's `biz` or `data`.** Only the exported interface
   at the module root. Also `depguard`-enforced.
5. **`reporting` is read-only and may read projections directly.** A deliberate exception
   — forcing every aggregate through per-module interfaces would generate an enormous
   amount of pass-through code for zero isolation benefit on a read-only path.

### Internal layering (go-kratos convention)

```
internal/module/sales/
├── sales.go          # exported interfaces + wire ProviderSet. The module's public face.
├── service/          # gRPC/HTTP handlers. Proto types ⇄ domain types. No business logic.
├── biz/              # domain types + use cases. NO proto imports, NO sql imports.
└── data/             # repository impls, sqlc queries, Redis, MinIO.
```

`biz` importing a generated proto type is the standard way a modular monolith rots — it
couples the domain model to the wire format, so every wire change becomes a domain change
and the domain stops being independently testable. `depguard` blocks `gen/go/...` imports
from any `biz` package.

Dependency injection is Google Wire, compile-time. Runtime DI containers trade a
build-time error for a startup-time panic, which is a bad trade for a binary that runs in
a store.

## 2.3 Transport

Both protocols are generated from the same `.proto`. REST is not a second API; it is a
second encoding of one API, via `google.api.http` annotations.

```protobuf
service SalesService {
  rpc GetSale(GetSaleRequest) returns (Sale) {
    option (google.api.http) = { get: "/v1/sales/{sale_id}" };
  }
}
```

| Consumer | Protocol | Why |
|---|---|---|
| PWA client | gRPC-Web (Connect) | Typed, binary, streaming-capable for sync pull |
| Third-party integrations | REST/JSON | No toolchain required of a partner |
| Internal tooling / admin scripts | gRPC | Typed |
| Webhooks out | REST/JSON | Universal |

**Connect** (`connectrpc.com`) is the recommended gRPC-Web implementation: it speaks
gRPC, gRPC-Web, and its own HTTP/JSON protocol from one handler, which removes the
grpc-gateway proxy hop and works in a browser without an Envoy sidecar.
See [ADR-0012](../decisions/0012-connect-grpc-web.md).

### Middleware chain

Order is load-bearing — recovery must be outermost so it catches panics from everything
inside it, and tracing must precede logging so logs carry a trace ID.

```
recovery → tracing → metrics → logging → rate limit → circuit breaker
        → auth (OIDC/JWT) → tenant resolution → validation → handler
```

Notes:

- **Validation** is `protoc-gen-validate` / `protovalidate` — rules declared in the proto,
  enforced identically on both sides of the wire. Structural validation only; business
  invariants live in `biz`.
- **Tenant resolution** extracts `tenant_id` from the verified token and puts it in
  context. Repositories read it from context; no query is written with a caller-supplied
  tenant ID. This is the single highest-value line of defence against cross-tenant leakage.
- **Rate limiting and circuit breaking** — see §2.7.

## 2.4 Persistence

**Postgres is the ledger of record.** Redis is a cache and a lock manager; if Redis is
wiped, nothing is lost. MinIO holds receipt PDFs, product images, and export artifacts.

Query layer is **sqlc** — typed Go from hand-written SQL. Deliberately not an ORM: this
system's hard queries are reporting aggregates and event-log scans, exactly where ORMs
produce query plans nobody predicted. Migrations are plain SQL, forward-only, applied by
`golang-migrate` in a job that runs before the new binary starts. Down-migrations are
not used in production; recovery is roll-forward or restore.
See [ADR-0013](../decisions/0013-sqlc-not-orm.md).

### Two write models, and knowing which you are in

| | Operational | Master data |
|---|---|---|
| Examples | sales, payments, stock movements, cash movements | products, prices, customers, users |
| Written by | device, offline-capable | back office, online-only |
| Storage | append-only event table + projections | ordinary mutable rows |
| Concurrency | none possible — events are distinct facts | optimistic, `version` column, 409 on mismatch |
| Deletion | never; reversed by a compensating event | soft delete |

This split is the core of the design and also its main source of conceptual load. A
developer must know which side of the line they are on. Mitigation: the two live in
visibly different packages, use different repository base types, and the event tables are
physically `REVOKE UPDATE, DELETE` for the application role — so the wrong instinct fails
loudly at the database rather than silently at review.

See [ADR-0005](../decisions/0005-event-sourced-writes.md).

### Event storage

```sql
CREATE TABLE domain_events (
  event_id        uuid PRIMARY KEY,              -- client-generated; idempotency key
  tenant_id       uuid NOT NULL,
  store_id        uuid NOT NULL,
  device_id       uuid NOT NULL,
  device_seq      bigint NOT NULL,               -- monotonic per device; gap = data loss
  event_type      text NOT NULL,
  payload         bytea NOT NULL,                -- serialised protobuf
  payload_type    text NOT NULL,                 -- fully-qualified proto message name
  occurred_at     timestamptz NOT NULL,          -- device clock. UNTRUSTED.
  received_at     timestamptz NOT NULL DEFAULT now(),  -- server clock. Authoritative.
  UNIQUE (device_id, device_seq)
);
CREATE INDEX ON domain_events (tenant_id, store_id, received_at);
```

Three deliberate details:

- `event_id` is generated by the *client*. That is what makes retry-on-timeout safe: a
  device that never saw an ack re-pushes the same event and the server's `ON CONFLICT DO
  NOTHING` makes it a no-op. Without client-generated IDs, every flaky-network retry is a
  potential double-charge — and flaky networks are this product's normal condition.
- `occurred_at` and `received_at` are both stored and are never conflated. Device clocks
  on cheap tablets drift and are user-settable; `occurred_at` is what the shift report
  shows the staff, `received_at` is what any correctness argument is built on.
- `UNIQUE (device_id, device_seq)` makes loss detectable. A gap in a device's sequence is
  an alertable event, not something discovered during a month-end audit.

Payload is serialised Protobuf rather than JSONB: it is the same schema the wire uses,
Buf guards its evolution, and unknown-field preservation means an older server does not
destroy fields written by a newer client. The cost is that ad-hoc SQL inspection of
payloads is not possible — accepted, because projections are what people actually query,
and a small `pos-events decode` CLI covers debugging.

### Projections

Updated **in the same transaction** as the event insert. Not eventually consistent, not
via an async worker. A cashier who completes a sale and immediately opens the shift
report must see that sale. Async projection here buys throughput this system does not
need and costs a class of bug ("the number was wrong for four seconds") that destroys
trust in a money product.

Rebuilding a projection from the event log is a supported, tested operation — it is the
recovery path for a projection bug, and it is what makes the event log worth having.
See [`docs/runbooks/rebuild-projections.md`](../runbooks/rebuild-projections.md).

## 2.5 The `sync` module

The device-facing contract. Deliberately its own module so that no domain module has to
know about offline concerns.

```protobuf
service SyncService {
  // Idempotent. Retryable. Partial success is normal and expected.
  rpc PushEvents(PushEventsRequest) returns (PushEventsResponse);

  // Server-streaming delta pull from a cursor.
  rpc PullChanges(PullChangesRequest) returns (stream ChangeSet);

  // What this device is entitled to hold, per ADR-0004.
  rpc GetReplicationPolicy(GetReplicationPolicyRequest) returns (ReplicationPolicy);
}
```

`PushEvents` returns a **per-event** result, never a single overall status:

```protobuf
message EventResult {
  string event_id = 1;
  enum Status {
    ACCEPTED   = 0;  // persisted
    DUPLICATE  = 1;  // already had it — treat exactly as ACCEPTED, drain it
    REJECTED   = 2;  // permanently invalid; will never succeed; needs human attention
    DEFERRED   = 3;  // transient (dependency not yet synced); retry later
  }
  Status status = 2;
  string reason = 3;
  repeated FieldViolation violations = 4;
}
```

The four-state result is the crux of the whole protocol. A binary success/failure forces
the client to choose between infinitely retrying an event that can never succeed, and
discarding money. `REJECTED` moves an event to a quarantine queue that surfaces in the UI
as a manager task; `DEFERRED` retries with backoff.

**Rejection is a last resort.** A device that has been offline for two days is pushing
events made in good faith against a world it could not see. The server's job is to accept
them and record reality — including a sale of stock that turned out not to exist. Reject
only for: bad signature/auth, schema violation, or a tenant/store the device may not
write to. Never reject for a business-rule violation that has already happened in the
physical world.

`PullChanges` is cursor-based (`(tenant, store, table, watermark)`), not timestamp-based.
Timestamp cursors lose rows when two transactions commit out of order across a clock
boundary. Watermarks use a per-table monotonic `xmin`-style sequence.

## 2.6 Search

Postgres `tsvector`/`tsquery` for product and customer search. A generated `tsvector`
column with a GIN index, plus `pg_trgm` for typo tolerance on barcodes and SKUs, which
matters more at a till than semantic relevance does.

This is right for the actual query: a cashier types three characters of a product name
and needs the top ten from a catalog of thousands, scoped to one store. That is not a
search-engine problem. Adding Elasticsearch would add a stateful service, a sync pipeline,
and a new class of "search is stale" bug for no user-visible gain at this size.

It stops being right when: catalogs exceed roughly a million rows per tenant, when
cross-tenant or cross-store federated search appears, when relevance tuning becomes a
product feature, or when faceted navigation is required. See
[ADR-0008](../decisions/0008-postgres-fts.md) for the trigger conditions and the exit path.

Note that offline search does **not** use any of this — it runs client-side against the
local store. The two implementations must not drift in a way that ranks results
differently online and offline, because a cashier will notice. Shared test corpus, one
golden-file test suite run against both.

## 2.7 Resilience

Kratos ships middleware for most of this; the decision is the policy, not the mechanism.

| Concern | Mechanism | Policy |
|---|---|---|
| Rate limit (per device) | Kratos `ratelimit` + Redis token bucket | Generous on `PushEvents` — a device draining two days of backlog is legitimate traffic and must not be throttled into a permanent backlog |
| Rate limit (per tenant) | Redis | Protects noisy-neighbour blast radius |
| Circuit breaking | Kratos `circuitbreaker` (sre.Breaker) | Only around genuinely external calls: payment gateway, email, webhooks |
| Timeouts | Per-endpoint, from config | Default 5 s; `PushEvents` 30 s; report exports 120 s |
| Retry | Client-side only, exponential + full jitter | Server never retries a write on a caller's behalf |
| Bulkhead | Separate worker pool for reporting | Stops a month-end export starving the till path |
| Graceful degradation | Feature flags (Unleash) | Kill-switch per non-essential subsystem |

**Circuit breakers do not belong between internal modules.** They are in-process function
calls; a breaker there converts a bug into a different bug. This is worth stating
explicitly because "add resilience middleware everywhere" is a common reflex.

**The till path is the priority queue.** If the server is degraded, `PushEvents` and
`PullChanges` are the last things to be shed. Reporting, exports, and webhooks are shed
first, by flag.

## 2.8 Testing

| Layer | Tool | Rule |
|---|---|---|
| `biz` | `testing` + Testify, table-driven | Pure, no I/O, fast. Where money maths is proven. |
| `data` | Testcontainers-Go (real Postgres) | No sqlmock — it tests the mock, not the query |
| `service` | in-process gRPC, faked `biz` | Wire mapping and error codes |
| Contract | Buf breaking + generated-client round-trip | Runs on every proto change |
| Sync | Scripted partition scenarios against real Postgres | See below |
| Migrations | Apply to a restored prod-shaped dump | Catches lock/table-rewrite surprises |

The sync scenario suite is the one that earns its keep. Minimum cases, all of which are
real incidents waiting to happen:

- Device offline 48 h, 2 000 queued events, pushes all at once
- Two devices sell the same last unit while both offline
- Duplicate push after ack lost in flight (must be idempotent, must not double-count)
- Device clock set two days into the future
- Device clock set backwards mid-shift
- Push interrupted at event 500 of 2 000; resume must not duplicate or skip
- Shift closed on device A while device B still has undrained events
- Event referencing a product created on another device that has not synced yet (`DEFERRED`)

Coverage target is 100% of `biz` branches per project standards. `data` and `service`
follow, but the honest statement is that branch coverage on a repository layer is a much
weaker signal than one scenario test that partitions the network.

---

## Tradeoffs

**Modular monolith vs. services.** We get one transaction, one deploy, one place to look
during an incident, and no distributed-tracing archaeology to answer "where did this sale
go." We give up independent scaling and independent deploy cadence, and we accept that
the boundaries are only as real as `depguard` and code review make them. The specific
risk is boundary erosion under deadline: someone adds a cross-module `JOIN` because it is
four lines instead of forty. That is why the rule is machine-enforced. **Revisit when:**
one module's resource profile diverges sharply (reporting needing 10× the memory), a team
grows past roughly 8–10 engineers and deploy contention becomes real, or one module needs
a genuinely different availability guarantee.

**Extraction is not free even with good boundaries.** Extracting `reporting` means
cross-process reads and eventual consistency; extracting `sales` means the atomic
transaction is gone and you are writing a saga. Realistically `reporting` and `identity`
are extractable, and `sales`+`inventory`+`shifts` are one unit approximately forever.
Saying so now is more useful than a diagram implying everything is equally extractable.

**Synchronous projections cost write latency** — a sale writes an event plus three or four
projection updates in one transaction. At POS volumes this is nowhere near a problem. It
would become one at thousands of writes per second per store, which is not a retail
number. Accepted without reservation.

**Protobuf payloads in Postgres are opaque to SQL.** A DBA cannot answer "which sales had
a manual discount?" by querying the payload. They query the projection instead, which is
where that field is materialised anyway. The residual pain is genuine during incident
forensics; the `pos-events decode` CLI is the mitigation and should be built early, not
when it is first needed at 3am.

**sqlc means writing SQL.** Slower for simple CRUD than an ORM. Faster and far more
predictable for the aggregates that dominate this system. Onboarding cost is real for
developers who arrived via ORMs.

**No internal event bus** keeps debugging simple and couples modules more tightly in time.
If a future module needs to react to `SaleCompleted` asynchronously (loyalty accrual,
external accounting push), it reads the event table with its own cursor — the log is
already there. That is the growth path, and it does not require adding a broker.

## Open Questions

1. **Tenancy isolation** (also raised in §1) — shared schema + RLS vs. schema-per-tenant.
   Blocks the first migration.
2. **Service-to-service auth.** With a single binary there is nearly no S2S surface today.
   Options evaluated in [ADR-0009](../decisions/0009-service-to-service-auth.md);
   recommendation is short-lived OpenBao-issued service tokens now, SPIFFE/mTLS when a
   second deployable exists. Confirm the team accepts deferring mTLS.
3. **Payment gateway integration.** Determines what "offline sale" means for card
   payments, which determines whether the offline story is real for a majority of
   transactions. Highest-value unknown in the entire document.
4. **Event retention.** Events are forever by default. Financial retention is typically
   7–10 years by statute. Need a partitioning strategy (monthly `received_at` partitions)
   and an archive-to-MinIO policy before the first table gets large.
5. **Do we need an outbox on the server** for webhooks/external pushes? Probably yes, same
   pattern, deferred until the first external integration is specified.
