# ADR-0002: Protobuf via Buf as the single source of truth for all contracts

## Status
Proposed

## Date
2026-08-11

## Context

Contracts cross three boundaries in this system:

1. Go server ⇄ TypeScript client (gRPC-Web and REST)
2. Server ⇄ third-party integrations (REST)
3. **Client outbox ⇄ server ingest ⇄ Postgres event store**

The third is unusual and decisive. Offline domain events are serialised on the device,
may sit in an outbox for days, and are then persisted permanently. An event written by a
2026 client must remain decodable by a 2033 server. Contract compatibility here is not
just an API concern — it is a data-durability concern.

The failure mode this project is most exposed to is drift between the Go and TypeScript
understanding of the same structure.

## Decision

Protobuf, managed with Buf, is the source of truth for every contract: API messages and
services, offline event payloads, validation rules (`protovalidate`), pricing rule
definitions, and the replication policy.

- Protos live in `api/` at the repo root — a sibling of `apps/`, owned by neither language.
- `buf generate` produces Go and TypeScript into `gen/`, which is committed and never
  hand-edited.
- CI runs `buf lint`, `buf format --diff --exit-code`, `buf breaking --against main`, and
  a regenerate-and-diff job.
- Versioning is by package path (`pos.sales.v1`); breaking changes require `v2` served in
  parallel.
- Zod schemas for client forms are generated from `protovalidate` options by a small
  in-repo generator, so client-side validation cannot drift from server-side validation.

## Alternatives Considered

### OpenAPI-first, JSON over REST
- Pros: universal, human-readable, no toolchain for partners, easy debugging.
- Cons: codegen quality varies sharply by language; no wire-level compatibility guarantees
  for stored events; field renames silently break consumers; no equivalent of Buf's
  breaking-change detection with comparable rigour.
- Rejected: acceptable for the API surface, inadequate for a durable event store.

### GraphQL
- Pros: excellent client-driven fetching; strong tooling.
- Cons: solves over-fetching, which a local-first client does not have — it reads from
  IndexedDB. Adds a query-complexity attack surface and does not address event persistence.
- Rejected: solves a problem this architecture does not have.

### JSON Schema, JSONB event payloads
- Pros: SQL-inspectable payloads; no codegen step.
- Cons: no enforced compatibility; larger storage; type drift between languages; nothing
  preventing a field's meaning changing under a live event store.
- Rejected: the SQL-inspectability benefit is real but is recovered via projections plus a
  decode CLI, at far lower risk.

### Protobuf without Buf (raw `protoc`)
- Pros: no extra tool.
- Cons: no lint, no breaking-change detection, no formatting, painful dependency
  management, plugin version drift across machines.
- Rejected: breaking-change detection is the single most valuable property here.

## Consequences

- Go and TypeScript types cannot drift; a mismatch is a build failure.
- Buf's breaking-change detection protects both live API compatibility and the
  decodability of years of stored events.
- **Field numbers may never be reused.** `reserved` on removed fields and names is
  mandatory. A reused field number silently misinterprets historical events.
- Every change costs a proto edit plus regeneration before feature work can start.
- REST is generated, not designed. `oneof` and well-known types map awkwardly to JSON; if
  a partner-facing REST API becomes a product surface it needs a hand-designed façade.
- Buf cannot detect *semantic* changes — a field whose meaning changes while its type
  stays passes every check. Only review catches this, and green CI creates false
  confidence.
- API docs (Scalar, Postman) are generated from the same source and cannot rot.

## Related
- [ADR-0001](0001-modular-monolith.md), [ADR-0005](0005-event-sourced-writes.md),
  [ADR-0012](0012-connect-grpc-web.md)
- [Architecture §5](../architecture/05-contracts.md)
