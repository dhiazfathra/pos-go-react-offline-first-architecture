# ADR-0009: Service-to-service auth — OpenBao-issued short-lived tokens now, SPIFFE/mTLS later

## Status

Proposed

## Date

2026-08-11

## Context

The stack constraints require evaluating mTLS versus internal service tokens versus
workload identity, and recommending one.

The relevant fact is that today there is **one application deployable**
([ADR-0001](0001-modular-monolith.md)). The "service-to-service" surface is:

| From | To | Nature |
|---|---|---|
| Server | Postgres | Long-lived connection pool, credential |
| Server | Redis | Connection, credential |
| Server | MinIO | S3 API, access key |
| Server | Unleash | HTTP, API token |
| Server | OpenBao | AppRole/JWT auth to obtain the above |
| Server | Payment gateway | External, provider-defined auth |

Cross-module calls inside the monolith are in-process Go function calls. There is no
network hop and therefore nothing to authenticate.

## Decision

**Now: short-lived, dynamically issued credentials from OpenBao. No service mesh.**

- The server authenticates to OpenBao once, via AppRole (or workload JWT where the platform
  provides one).
- Postgres credentials are **dynamic**, with a short TTL, issued per instance and rotated
  automatically. No static database password exists anywhere.
- MinIO, Redis and Unleash credentials are stored in OpenBao and injected at startup, never
  in environment files or IaC state.
- TLS terminates at the gateway. Internal traffic runs on an isolated network, and network
  isolation is treated as defence in depth, not as a substitute for encryption in transit.
- **Every stateful dependency is reached over TLS with certificate validation on** —
  Postgres (`sslmode=verify-full`), MinIO, Redis, Unleash, and OpenBao itself. Certificates
  come from OpenBao PKI; the CA bundle is pinned and `InsecureSkipVerify` is never set, in
  any environment. There is no per-dependency exception; a dependency that cannot present a
  valid certificate is not deployed until it can.
- **Credentials are renewed, not merely issued.** The OpenBao token is renewed on a timer
  well inside its TTL, and re-authenticates via AppRole from scratch if renewal fails. The
  Postgres lease is renewed the same way; when a lease cannot be extended, the server builds
  a replacement pool on freshly issued credentials, health-checks it, swaps it in
  atomically, and drains the old pool rather than severing in-flight transactions.
- External calls (payment gateway) use the provider's mechanism plus certificate pinning
  where supported.

**Later: SPIFFE/SPIRE workload identity with mTLS**, adopted at the first of:

- a second application deployable exists (first extracted service), or
- a compliance requirement mandates encryption in transit between internal workloads, or
- the platform moves to Kubernetes, where SPIFFE integration is largely free.

## Alternatives Considered

### Service mesh with mTLS now (Istio / Linkerd)

- Pros: encryption in transit and workload identity by default; policy in one place; a
  strong story for a security questionnaire.
- Cons: deploying a mesh to secure a single binary is infrastructure with no corresponding
  risk reduction. Sidecars, control plane, certificate rotation, and a new class of
  debugging — all real ongoing operational cost against a nearly empty threat model.
- Rejected **for now**, explicitly not forever. This is a sequencing decision.

### Static internal service tokens

- Pros: trivial to implement.
- Cons: long-lived shared secrets, distributed by hand, rotated rarely, and typically found
  in a repository some time later.
- Rejected: strictly worse than dynamic credentials for the same effort.

### SPIFFE/SPIRE now, without a mesh

- Pros: the correct end state; removes credential distribution entirely.
- Cons: SPIRE server, agents, and attestation to run — for one workload whose identity is
  not in question.
- Rejected now, adopted at the trigger conditions above.

### Cloud provider IAM roles

- Pros: no credential distribution at all where available.
- Cons: ties the platform to one cloud, and the deployment topology is not yet decided
  (§6 Open Question 1) — on-prem deployment is a live possibility.
- Rejected until the topology is settled. If the answer is single-cloud, this becomes the
  strongest option and this ADR should be superseded.

## Consequences

- No static database credentials. Compromise of the application host yields a credential
  that expires shortly.
- Operational dependency on OpenBao: if it is sealed or unreachable, new instances cannot
  obtain credentials and cannot start. Running instances continue until their lease
  expires. **Unsealing strategy is a blocking open question** and belongs in the DR runbook.
- The failure mode during an OpenBao outage is specific and worth stating plainly: existing
  Postgres sessions keep working until their lease expires, new connections cannot be
  opened, and the server degrades to serving what its current pool can carry rather than
  failing outright. Lease TTLs are therefore chosen to exceed the expected OpenBao recovery
  time, and lease expiry under outage is a scenario the DR rehearsal exercises deliberately
  — it is the one path where a credential system takes down an otherwise healthy database.
- No in-cluster mTLS today. This will appear on security questionnaires; the honest answer
  is single-workload plus network isolation plus TLS to every stateful dependency.
- Adding SPIFFE later is additive: it does not invalidate the OpenBao integration, which
  continues to issue database credentials.
- The security review at the first service extraction must revisit this ADR as a gate, not
  as a follow-up ticket.

## Related

- [ADR-0001](0001-modular-monolith.md), [ADR-0011](0011-podman-prod-docker-local.md)
- [Architecture §6.5](../architecture/06-platform.md#65-service-to-service-authentication)
