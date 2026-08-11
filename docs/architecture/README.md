# Architecture

Read in order. Each section ends with its own **Tradeoffs** and **Open Questions**.

| # | Section | Covers |
|---|---|---|
| [01](01-overview.md) | Overview | System boundaries, framing, monorepo layout, data flow, environments |
| [02](02-server.md) | Server | Module boundaries, transport, persistence, sync module, search, resilience, testing |
| [03](03-client.md) | Client | TanStack Start structure, data layer, outbox, write path, performance, keyboard, testing |
| [04](04-sync.md) | Local-first sync | Offline scope, replication tiers, write path, conflicts, protocol, receipt numbering, duplicated domain logic |
| [05](05-contracts.md) | Shared contracts | Protobuf layout, codegen, consumption, versioning, contract testing |
| [06](06-platform.md) | Platform | Runtime, IaC, CI/CD gates, auth, observability, feature flags, docs standards |

## The one-paragraph version

A multi-tenant POS platform. Go modular monolith (go-kratos) behind gRPC + REST, both
generated from Protobuf managed by Buf. React/TanStack Start PWA that reads exclusively
from local storage and writes exclusively to a durable outbox. Postgres is the ledger of
record. Operational writes are immutable domain events; master data is ordinary CRUD.
Stock on hand is a projection, never a written field.

## The framing that matters

**POS is an offline event-sourcing problem with a cached read model on the side** — not a
replicated-document problem.

The local-first playbook (Linear-style) contributes its *client tactics*: optimistic local
writes, granular reactivity, local search, code splitting, service-worker precache. It does
not contribute its *architecture premise*: the server here validates rather than merges,
Postgres stays authoritative, and the client is a scoped cache plus an outbox rather than a
peer replica.

Why: a lost write in a document app is an annoyance; in a POS it is money that cannot be
reconciled. See [§1.2](01-overview.md#12-the-framing-that-drives-the-design).

## Decisions

See [`../decisions/`](../decisions/). The three carrying the most weight are
[ADR-0005](../decisions/0005-event-sourced-writes.md),
[ADR-0003](../decisions/0003-offline-scope.md), and
[ADR-0006](../decisions/0006-oversell-accepted.md).

## Status

Proposed. No code exists yet. Several open questions block implementation — the highest
priority are collected in [`../README.md`](../README.md#blocking-questions).
