# ADR-0001: Go modular monolith with go-kratos

## Status

Proposed

## Date

2026-08-11

## Context

The backend must serve a multi-tenant, multi-store POS platform: catalog, pricing,
inventory, sales, shifts, customers, identity, reporting, and a device-facing sync
endpoint.

The dominant constraint is transactional. Completing a sale must atomically write a domain
event, update inventory movements, update shift totals, and update read projections. If
these live in different services, that transaction becomes a distributed saga with
compensating actions — and a partially-compensated sale is an unreconcilable till.

Secondary constraints: a small team, one deployable to operate, and a stack fixed to Go.

## Decision

A modular monolith in Go, using go-kratos, deployed as a single binary against a single
Postgres database.

Modules are bounded contexts (`identity`, `catalog`, `pricing`, `inventory`, `sales`,
`shifts`, `customers`, `reporting`, `sync`). Each has `service` / `biz` / `data` layers and
exposes a Go interface at its root.

Boundaries are enforced by tooling, not convention:

- `depguard` in `golangci-lint` blocks cross-module imports of `biz` and `data`.
- `depguard` blocks `gen/go` (proto) imports from any `biz` package.
- No cross-module table access; consumers declare the narrow interface they need and the
  producing module satisfies it.
- No internal message bus. Cross-module calls are synchronous in-process interface calls.

## Alternatives Considered

### Microservices from the start

- Pros: independent scaling and deploy; strong boundaries by construction; conventional.
- Cons: the sale transaction becomes a saga; N deployables for a small team; distributed
  tracing needed to answer basic questions; network failure modes added to a system whose
  entire premise is coping with network failure.
- Rejected: it distributes the one thing that must not be distributed, and buys scaling we
  do not need at a cost we cannot afford.

### Single-package monolith ("start simple, extract later")

- Pros: fastest initial velocity, no boundary ceremony.
- Cons: without enforced boundaries, extraction is never actually possible; the promised
  "later" arrives as a rewrite.
- Rejected: the boundary cost is small and paid once; the un-boundaried version compounds.

### Monolith with an internal event bus between modules

- Pros: looser coupling; a ready-made path to extraction.
- Cons: asynchrony inside one process, harder debugging, eventual consistency where none
  is required.
- Rejected: interfaces give most of the decoupling at a fraction of the cost. Where async
  reaction is later needed, the persisted event log already supports a cursor-reading
  consumer without adding a broker.

### Go standard library / chi, no framework

- Pros: fewer dependencies; total control.
- Cons: we would hand-roll middleware chains, config, DI wiring, and gRPC+REST duality.
- Rejected: go-kratos supplies exactly this, is Protobuf-first, and matches the stack
  constraint.

## Consequences

- One transaction covers a sale. No sagas, no compensating actions.
- One deployable, one place to look during an incident.
- `depguard` config is load-bearing. It must be maintained, and a bypass must fail CI.
- Boundary erosion under deadline pressure is the main risk. Machine enforcement is the
  mitigation; review is the backstop.
- Extraction is not uniformly available. `reporting` and `identity` are realistically
  extractable. `sales` + `inventory` + `shifts` are one transactional unit and are best
  understood as permanently co-located.
- Google Wire gives compile-time DI: wiring mistakes are build errors, not startup panics.

## Revisit when

- One module's resource profile diverges sharply (e.g. reporting needing 10× memory).
- Team size passes roughly 8–10 engineers and deploy contention becomes real.
- One module requires a different availability or compliance guarantee.

## Related

- [ADR-0002](0002-protobuf-buf-contracts.md), [ADR-0013](0013-sqlc-not-orm.md)
- [Architecture §2](../architecture/02-server.md)
