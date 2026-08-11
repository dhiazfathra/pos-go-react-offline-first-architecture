# ADR-0011: Podman with quadlets in production, Docker locally

## Status

Proposed

## Date

2026-08-11

## Context

The stack constraint specifies Podman in staging and production, Docker locally. This is
a deliberate inconsistency and deserves a written rationale, because "works on my machine"
is precisely the failure it invites.

The case for each:

**Podman in production** — daemonless and rootless. There is no long-lived root daemon
whose compromise owns the host, which is a genuine reduction in attack surface for a system
processing payment data. Containers run as systemd units (quadlets), so they integrate with
the service manager already supervising the host, and get restart policies, journald
logging and dependency ordering for free.

**Docker locally** — the ecosystem assumes it. Testcontainers, Compose, IDE integrations
and most third-party tooling target the Docker socket. Rootless Podman on macOS adds a VM
layer and a set of papercuts that cost developer time every day.

## Decision

- **Local dev and CI:** Docker / Docker Compose / Testcontainers.
- **Staging and production:** Podman with quadlets, rootless.

Mitigations, which are the substance of this decision:

1. **Build once, promote unchanged — by digest.** Images are OCI, built once in CI, and
   promoted through environments without rebuilding. The commit-SHA tag is for humans; the
   `sha256:` **digest is what is promoted and deployed**, because a tag is a mutable pointer
   a registry can retarget and therefore guarantees nothing about content. Quadlets pin
   `image=…@sha256:…`, deployment compares the digest actually running against the digest CI
   produced and fails the rollout if they differ, and images are signed with provenance
   attestation verified at pull. This is what makes "byte-identical to the one that passed
   CI" a checkable claim rather than an assumption.
2. **No `podman-compose`.** Production uses quadlets directly, so the production
   orchestration path is exercised as itself rather than through a compatibility shim that
   hides differences.
3. **Staging is Podman, and every release passes through staging.** The Podman path is
   exercised before production on every single change — this is what makes the
   inconsistency survivable.
4. **Rootless-specific constraints are documented and encoded**: UID/GID mapping, volume
   ownership, ports below 1024, and cgroup differences live in
   `docs/runbooks/deploy.md` and in the quadlet templates, not in someone's memory.
5. **Containers are non-root in both environments.** The Dockerfile sets `USER`, so the
   rootless behaviour is not a production-only surprise.

## Alternatives Considered

### Docker everywhere

- Pros: complete consistency; simplest to reason about.
- Cons: forfeits the daemonless/rootless security benefit in production, which is the
  reason Podman was specified.
- Rejected: consistency is valuable but is recoverable through OCI portability; the daemon
  attack surface is not.

### Podman everywhere, including local

- Pros: true dev/prod parity.
- Cons: Testcontainers requires additional configuration; macOS needs `podman machine`;
  various tools assume the Docker socket. A daily tax on every developer to remove a risk
  already contained by build-once-promote and by staging.
- Rejected now. **This is the first thing to revisit** if a Podman-specific defect ever
  reaches production — that event would invalidate the reasoning above.

### Kubernetes in production

- Pros: rolling deploys, autoscaling, huge ecosystem, standard operational patterns.
- Cons: a control plane to operate for a small number of nodes and one application;
  substantially more moving parts than the workload warrants.
- Rejected at current scale. Revisit at multi-region, at roughly ten deployables, or when
  the team would rather operate Kubernetes than systemd.

## Consequences

- Production has no container daemon running as root.
- systemd supervises containers: restart policy, ordering, journald, and `systemctl` for
  operators who already know it.
- Two orchestration definitions exist — Compose for local, quadlets for production. They
  will drift. Mitigation is that local Compose is for *dependencies* (Postgres, Redis,
  MinIO), while the application in production is deployed by quadlet; the overlap is
  deliberately small.
- Rootless networking differs (no bridge by default, port mapping via slirp4netns or
  pasta). Documented, and encoded in the templates.
- Volume ownership under UID mapping is the most common operational surprise. It is called
  out in the deploy runbook.
- Testcontainers in CI stays on Docker; no attempt is made to run it under Podman.

## Related

- [ADR-0009](0009-service-to-service-auth.md)
- [Architecture §6.1](../architecture/06-platform.md#61-runtime),
  [`docs/runbooks/deploy.md`](../runbooks/deploy.md)
