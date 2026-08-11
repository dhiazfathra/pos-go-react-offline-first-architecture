# POS — offline-first point-of-sale platform

Multi-tenant, multi-store point of sale. Stores keep taking money when the network is gone,
and every cent taken is reconcilable afterwards.

**Status: design. No implementation yet.** This repository currently contains the
architecture and the decision records. See [`docs/`](docs/).

## The shape of it

```
Device (installed PWA)  ──▶  local store + durable outbox
        │                          (all reads and writes are local)
        │ sync engine: push events, pull deltas
        ▼
Gateway  ──▶  Go modular monolith (go-kratos), gRPC + REST
        ──▶  Postgres (ledger of record) · Redis · MinIO
```

Operational writes are immutable domain events appended on the device and validated
server-side. Master data is ordinary CRUD, online-only. Stock on hand is a projection of
the movement log, never a written field.

## Device requirement

Tills run a Chromium-based browser (Chrome or Edge) on Android, Windows, or ChromeOS.
iOS/iPadOS is **not** a supported till platform as designed: Safari has no Web
Serial/USB/Bluetooth, so receipt printers and card terminals cannot be driven from the PWA.
This is a procurement constraint stated up front, not something to discover during
rollout — see [ADR-0010](docs/decisions/0010-pwa-not-native.md).

Two things are still open and must be answered before building: whether iOS tills are a
hard requirement (if so, ADR-0010 fails and a native shell becomes mandatory —
[docs/README.md](docs/README.md#blocking-questions)), and the exact minimum browser
version, which fixes `esnext` targeting and `storage.persist()` behaviour
([03-client Open Questions](docs/architecture/03-client.md#open-questions)).

## Stack

| Layer | Choice |
|---|---|
| Backend | Go, go-kratos, modular monolith |
| Contracts | Protobuf, Buf (lint, breaking-change detection, codegen) |
| Protocols | gRPC (Connect) + REST (grpc-gateway) |
| API docs | Scalar, Postman — both generated |
| Frontend | React, TanStack Start, TanStack Query, React Hook Form + Zod, Tailwind, shadcn/ui |
| Mobile | PWA — same codebase, no native app |
| Database | PostgreSQL, with `tsvector` full-text search |
| Cache | Redis |
| Object storage | MinIO |
| Data access | sqlc, forward-only SQL migrations |
| IaC / secrets | OpenTofu, OpenBao |
| Runtime | Podman quadlets (staging/prod), Docker (local) |
| CI/CD | GitHub Actions |
| Observability | Prometheus, Grafana, Loki, Jaeger, Sentry, PostHog |
| Feature flags | Unleash |
| Testing | Go `testing` + Testify + Testcontainers; Vitest + Playwright |
| Quality | golangci-lint, ESLint, Prettier, Lefthook, SonarQube |

## Documentation

| | |
|---|---|
| [Architecture](docs/architecture/) | How it works and why. Read [01-overview](docs/architecture/01-overview.md) first. |
| [Decisions](docs/decisions/) | ADRs. Start with [0005](docs/decisions/0005-event-sourced-writes.md). |
| [Runbooks](docs/runbooks/) | Deploy, rollback, sync troubleshooting, DR. |
| [Open questions](docs/README.md#blocking-questions) | What must be answered before building. |

## Quick start

Not yet applicable — no code. Once `apps/server` and `apps/web` exist this section covers
prerequisites, `docker compose up` for dependencies, `buf generate`, and the dev servers.

## Commands

| Command | Description |
|---|---|
| — | To be added with the first code |

## Contributing

- Every architectural decision gets an ADR. Template and process:
  [`docs/decisions/README.md`](docs/decisions/README.md).
- Protobuf is the source of truth. Never hand-edit `gen/`.
- Quality gates are in [§6.3](docs/architecture/06-platform.md#63-cicd) and are blocking.
- Conventional commits.
