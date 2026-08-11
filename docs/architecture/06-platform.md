# 6. Platform — Infrastructure, Delivery, Security, Observability

## 6.1 Runtime

| Environment | Runtime | Rationale |
|---|---|---|
| Local dev | Docker Compose | Ubiquitous, best DX, matches what every contributor already has |
| CI | GitHub Actions + Testcontainers | Docker socket assumed by the ecosystem |
| Staging / Production | Podman + quadlets (systemd units) | Daemonless, rootless, no privileged long-lived daemon |

The inconsistency is real and deliberate. Podman's rootless, daemonless model is a genuine
security improvement for a system holding payment data — there is no root daemon whose
compromise owns the host. Docker's developer experience and ecosystem assumptions
(Testcontainers, Compose, IDE integration) are meaningfully better locally.

Mitigations, because "it works on my machine" is exactly the failure this invites:

- Everything is OCI. Images are built once in CI and promoted unchanged through
  environments — never rebuilt per environment.
- `podman-compose` is **not** used. Production uses quadlets directly, so the production
  orchestration path is exercised as itself rather than through a compatibility shim.
- Staging is Podman and every release passes through staging. The Podman path is exercised
  before production on every single change.
- Rootless-specific constraints (UID mapping, volume ownership, port binding below 1024)
  are documented in `docs/runbooks/deploy.md` and encoded in the quadlet templates.

See [ADR-0011](../decisions/0011-podman-prod-docker-local.md), including the trigger for
revisiting (Kubernetes becomes justified at roughly the point of multi-region or
double-digit deployables).

## 6.2 Infrastructure as code

**OpenTofu**, layout `deploy/tofu/{modules,envs/{staging,prod}}`.

- Remote state with locking; state is itself a secret and is encrypted at rest.
- `tofu plan` runs on every PR touching `deploy/`; the plan is posted to the PR.
- `tofu apply` is manual-approval on `main` only, and never runs from a developer machine
  against production.
- No secret values in tofu. Only OpenBao *references*.

**OpenBao** for secrets. Dynamic Postgres credentials with short TTLs, transit encryption
for application-level field encryption, and PKI for internal certificates. The application
authenticates via AppRole/JWT and never carries a static database password.

The immediate operational question OpenBao raises is unsealing — auto-unseal requires a
cloud KMS or an HSM, and manual unseal means a human is required to restore service after
a restart. That belongs in the DR runbook and in Open Questions, because discovering it
during an incident is the wrong time.

## 6.3 CI/CD

GitHub Actions. Path-filtered so a proto change does not run the Playwright suite for no
reason, and vice versa.

```
PR
 ├─ detect changed paths
 ├─ proto:    buf lint · buf format · buf breaking · generate-and-diff
 ├─ go:       golangci-lint · go test -race -cover · Testcontainers suite
 ├─ web:      eslint · prettier · tsc --noEmit · vitest --coverage
 ├─ e2e:      playwright (incl. offline scenarios)
 ├─ security: govulncheck · pnpm audit · trivy image · gitleaks
 ├─ quality:  SonarQube gate
 └─ build:    OCI images, tagged by commit SHA
main
 ├─ promote image → staging (auto)
 ├─ smoke + migration verification
 └─ promote image → production (manual approval)
```

### Quality gates (blocking)

| Gate | Threshold |
|---|---|
| Coverage on new code | 100% (project standard) |
| Coverage overall | no decrease |
| SonarQube | 0 blocker/critical issues on new code |
| Duplication on new code | < 3% |
| `golangci-lint` | 0 issues, warnings included |
| ESLint | 0 errors, 0 warnings |
| `buf breaking` | 0 |
| Vulnerabilities | 0 high/critical in production dependencies |
| Secret scan | 0 |

**Lefthook** runs the fast subset pre-commit — format, lint changed files, `buf lint`. The
full suite stays in CI. A pre-commit hook that takes two minutes is a pre-commit hook that
gets bypassed with `--no-verify`, so keeping it under a few seconds is a design constraint,
not a nicety.

### Migrations

Run as a separate job before the new binary starts, and must be backward-compatible with
the *currently running* version — the two overlap during deploy. Practically: expand,
migrate, contract, across three releases for anything destructive. Forward-only. Rollback
is roll-forward or restore from PITR; there are no down-migrations in production.

## 6.4 Authentication and authorisation

**Users**: OAuth 2.0 / OIDC, short-lived access JWTs plus refresh tokens.

The POS-specific problem: **a token expiring mid-outage cannot be refreshed.** A cashier
must not be logged out because the internet has been down for three hours.

Resolution:

- Access token TTL is long enough to cover a plausible outage (12 h) and is scoped
  narrowly — a device token can push events for its own store and nothing else.
- Device identity is a separate, long-lived, revocable credential, bound to the device at
  registration. It is what authenticates `PushEvents`.
- The **cashier PIN is verified locally** against a cached Argon2id hash. It is an
  attribution and till-access control, not a security boundary — a device with physical
  access is already trusted to take money.
- Revocation is not real-time offline. A device revoked while offline continues to operate
  until it reconnects, at which point its queued events are still *accepted* (the sales
  happened) but the device is locked out. Accepting the queued events is deliberate:
  refusing them would destroy records of real money to punish a device, which is the wrong
  trade.

**Authorisation**: RBAC with roles per tenant (`owner`, `manager`, `supervisor`,
`cashier`), evaluated server-side on every request. Permissions are replicated to Tier 1
so the client can hide what a user cannot do — hiding is UX, not enforcement, and the
server never trusts it.

## 6.5 Service-to-service authentication

Evaluated in [ADR-0009](../decisions/0009-service-to-service-auth.md).

**Recommendation: short-lived OpenBao-issued service tokens now; SPIFFE/SPIRE workload
identity with mTLS when a second deployable exists.**

Reasoning: today there is exactly one application deployable. The S2S surface is the
monolith talking to Postgres, Redis, MinIO, Unleash and OpenBao — all of which take a
credential, none of which need a mesh. Deploying a service mesh to secure a single binary
is infrastructure with no corresponding risk reduction, and it is real ongoing operational
cost.

mTLS is not skipped, it is *placed*: TLS terminates at the gateway, internal traffic runs
on an isolated network, and infrastructure connections (Postgres, MinIO) use TLS with
certificates issued by OpenBao PKI.

Trigger for moving to SPIFFE/mTLS: the first extracted service, or the first compliance
requirement mandating in-cluster encryption in transit. Workload identity is the right end
state — it removes credential distribution entirely — and adopting it before there are
workloads to identify is premature.

## 6.6 Observability

| Signal | Tool | Notes |
|---|---|---|
| Metrics | Prometheus + Grafana | RED per endpoint; sync metrics per device (§4.10) |
| Logs | Loki | Structured, trace-correlated, PII-redacted |
| Traces | Jaeger, OpenTelemetry | Sampled; `PushEvents` always traced |
| Errors | Sentry | Server and client, release-tagged, source-mapped |
| Product analytics | PostHog | Feature usage, funnels, session replay on non-PII screens |

All instrumentation goes through OpenTelemetry so backends are swappable.

### What actually matters here

Generic dashboards will not catch this system's failures. The alerts that matter are in
[§4.10](04-sync.md#410-observability) and they are **per-device**: an aggregate sync-health
number hides the one store that has been dark since Tuesday.

Add to those:

- **Client-side error and performance reporting is buffered offline and sent on
  reconnect.** A crash during an outage is exactly the crash worth knowing about, and it
  is the one a naive Sentry integration silently discards.
- **PII redaction is enforced at the logging library**, not left to reviewers. Customer
  names, contacts, and payment identifiers never reach Loki. The proposed proto-level PII
  annotation (§5, Open Question 5) would let this be generated rather than remembered.

### SLOs (initial, to be revised against real data)

| SLO | Target |
|---|---|
| `PushEvents` availability | 99.9% |
| `PushEvents` p99 latency (200-event batch) | < 2 s |
| Sync freshness (device drained within) | 5 min at p95 during trading hours |
| Data loss | zero, alerted on any `device_seq` gap |

Availability of the *server* is deliberately not the headline SLO. Stores keep trading
when the server is down — that is the point of the architecture. The number that matters
is whether the events eventually arrive intact.

## 6.7 Feature flags

**Unleash**, self-hosted alongside the platform.

| Kind | Lifetime | Owner | Removal |
|---|---|---|---|
| Release | ≤ 60 days | Feature author | Mandatory; CI warns at 60 days, fails at 90 |
| Kill-switch | Permanent | Module owner | Reviewed quarterly |
| Experiment | ≤ 90 days | Product | Removed at conclusion |
| Permission | Permanent | Product | Not really a flag — prefer RBAC |

Rules:

- Every flag has an owner and an expiry recorded at creation. A flag with no owner is
  deleted.
- Flag state is **replicated to Tier 1** and evaluated locally. A till cannot call Unleash
  mid-outage, so the client evaluates against the last known state.
- **Consequence: flag changes are not instant on offline devices.** A kill-switch cannot
  save a device that is offline. Anything needing a guaranteed instant kill must be
  server-side, and the till path must be safe by default.
- Flags must never gate a financial calculation. A basket priced under flag A and synced
  under flag B is unreconcilable. Flags gate UI and non-financial behaviour only. This is
  a hard rule and worth enforcing in review.

## 6.8 Documentation standards

| Artifact | Location | Naming |
|---|---|---|
| ADR | `docs/decisions/` | `NNNN-kebab-title.md`, sequential, never renumbered |
| Architecture | `docs/architecture/` | `NN-topic.md` |
| Runbook | `docs/runbooks/` | `kebab-title.md` |
| Templates | `docs/templates/` | `*.template.md` |
| Package README | alongside the code | `README.md` |

**ADR process.** Author opens a PR with status `Proposed`. It needs one approval from a
maintainer plus one from anyone materially affected. Merged as `Accepted`. Never edited
afterwards except to change status — a superseding ADR is written instead, and links both
ways. ADRs are not deleted; a wrong decision with recorded reasoning is more useful than
no record.

**When an ADR is required:** any choice that would be expensive to reverse — a framework
or major dependency, a data model, an auth strategy, an API protocol, anything changing
the offline guarantee, and anything overriding a constraint in this document.

Templates live in [`docs/templates/`](../templates/).

---

## Tradeoffs

**Docker locally, Podman in production** — §6.1. Contained by build-once-promote and by
staging being Podman, not eliminated. The residual risk is a rootless-specific failure
appearing first in staging; that is an acceptable place to find it.

**Self-hosting Unleash, OpenBao, SonarQube, and the full observability stack** is a
substantial operational surface for a small team. Each is defensible alone; together they
are a part-time platform job. Worth a deliberate check on whether managed equivalents are
cheaper than the engineering time, particularly for observability.

**OpenBao unsealing** is an availability dependency. Auto-unseal needs a KMS, which
reintroduces a cloud dependency; manual unseal needs a human at 3am. Must be decided and
rehearsed.

**Long-lived access tokens (12 h)** widen the window a stolen token is useful. Accepted
because the alternative — a cashier locked out mid-outage — defeats the product. Narrow
scoping and device binding are the compensating controls.

**Offline revocation is not real-time.** A stolen device keeps working until it
reconnects. Compensating controls are physical device management and post-hoc audit. This
should be stated to the customer rather than implied away.

**100% coverage on new code** is the project standard and is genuinely valuable on domain
and sync logic. On presentational components it mostly measures that a component rendered
once. Expect pressure to relax it; the useful compromise is to keep the gate and be
deliberate about what lives in the excluded paths.

**No Kubernetes.** Quadlets are simpler and adequate for single-region, few-node
deployments. They give up autoscaling, rolling-deploy primitives, and the ecosystem.
Revisit at multi-region, at roughly ten deployables, or when the team hires someone who
would rather operate Kubernetes than systemd.

## Open Questions

1. **Deployment topology.** Single region? Per-tenant isolation? Cloud, or on-prem at
   large customers? On-prem changes IaC, observability egress, and the update strategy
   substantially.
2. **OpenBao unseal strategy** — see above. Blocks the DR runbook.
3. **Managed vs. self-hosted observability.** Loki + Jaeger + Prometheus at retention is a
   real storage and operations bill.
4. **PCI DSS scope.** If card data ever touches our systems — as opposed to a P2PE
   terminal — scope expands enormously and constrains logging, network segmentation, and
   access control. Needs a definitive answer early; it is cheaper to design out of scope
   than to descope later.
5. **Backup and DR targets.** RPO/RTO are undefined. For a POS the interesting figure is
   not server RTO — stores keep trading — but how much *ingested* data may be lost, which
   should approach zero given PITR.
6. **Device fleet management.** How are tablets enrolled, updated, locked down, and wiped
   when lost? Adjacent to this architecture but it determines whether the offline-revocation
   gap is acceptable.
