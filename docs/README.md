# Documentation

| Directory | Contents |
|---|---|
| [`architecture/`](architecture/) | How the system is designed, and why |
| [`decisions/`](decisions/) | ADRs — decisions expensive to reverse |
| [`runbooks/`](runbooks/) | Operational procedures |
| [`templates/`](templates/) | ADR, README, and runbook templates |
| [`superpowers/specs/`](superpowers/specs/) | Design specs that record how a decision was reached |

## Start here

1. [Architecture overview](architecture/01-overview.md) — the framing and the layout
2. [ADR-0005](decisions/0005-event-sourced-writes.md) — why a sale cannot conflict
3. [Sync strategy](architecture/04-sync.md) — the load-bearing section

## Blocking questions

Collected from every section. These need answers before implementation starts, roughly in
order of how much they would change.

| # | Question | Blocks | Owner |
|---|---|---|---|
| 1 | **Payment terminal integration model** — can card payments authorise offline at all? | What "offline sale" actually means for the customer. If most transactions are card and terminals are cloud-only, offline mode is cash-only and the product's value proposition changes. | Product |
| 2 | **Fiscal / regulatory requirements per jurisdiction** — certified devices, gapless signed sequences, real-time government reporting? | Mandated real-time reporting is close to incompatible with offline selling. Could invalidate the design. | Legal / Product |
| 3 | **Oversell sign-off** ([ADR-0006](decisions/0006-oversell-accepted.md)) | The entire conflict model rests on this being an accepted business outcome. | Business |
| 4 | **iOS tablets + thermal printers required?** ([ADR-0010](decisions/0010-pwa-not-native.md)) | If yes, PWA-only fails and a native shell is mandatory. | Product / Procurement |
| 5 | **Tenancy isolation model** — shared schema + RLS vs. schema-per-tenant | The first migration. | Engineering |
| 6 | **Returns window** | Tier-2 replication window and device storage budget. | Product |
| 7 | **Expected scale** — stores/tenant, devices/store, transactions/device/day | Replication window, storage budget, whether Postgres FTS holds. | Product |
| 8 | **Deployment topology** — cloud, on-prem, single or multi region | IaC, observability, update strategy, service-to-service auth. | Engineering / Sales |
| 9 | **OpenBao unseal strategy** | The DR runbook. Recovery cannot begin without it. | Platform |
| 10 | **PCI DSS scope** | Logging, network segmentation, access control. Cheaper to design out of scope than to descope. | Security |
| 11 | **RPO/RTO targets** | Backup strategy and rehearsal cadence. | Business / Platform |

## Known gaps in the design

Not open questions — known missing pieces that must be built.

- **Device re-push for a sequence range.** Without it, any database restore permanently
  loses sales that devices believe were delivered. Required before production launch. See
  [`runbooks/disaster-recovery.md`](runbooks/disaster-recovery.md#5-reconcile-devices--the-step-that-matters).
- **Manager reconciliation UX** for quarantined events and oversell reports. The human end
  of the sync protocol, easy to leave until it is urgent.
- **`pos-events decode` CLI.** Protobuf payloads are opaque to SQL; this is the forensics
  tool. Build it before the first incident, not during one.
- **Pricing golden corpus** (`api/testdata/pricing/`). The only thing preventing the Go and
  TypeScript pricing implementations from drifting.

## Conventions

| Artifact | Location | Naming |
|---|---|---|
| ADR | `docs/decisions/` | `NNNN-kebab-title.md` |
| Architecture | `docs/architecture/` | `NN-topic.md` |
| Runbook | `docs/runbooks/` | `kebab-title.md` |
| Package README | alongside the code | `README.md` |

Process for each is in [§6.8](architecture/06-platform.md#68-documentation-standards).
