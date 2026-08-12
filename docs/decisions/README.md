# Architecture Decision Records

Decisions that would be expensive to reverse. Each records the context, the decision, what
was rejected and why, and the consequences we accepted.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-modular-monolith.md) | Go modular monolith with go-kratos | Proposed |
| [0002](0002-protobuf-buf-contracts.md) | Protobuf via Buf as the single source of truth | Proposed |
| [0003](0003-offline-scope.md) | Operating surface offline, administrative surface online | Proposed |
| [0004](0004-replication-tiers.md) | Four replication tiers, scoped by access pattern | Proposed |
| [0005](0005-event-sourced-writes.md) | Event-sourced operational writes, LWW master data | Proposed |
| [0006](0006-oversell-accepted.md) | Offline oversell accepted and reported, not prevented | Proposed — needs business sign-off |
| [0007](0007-receipt-numbering.md) | Device-prefixed local sequence + server fiscal sequence | Proposed |
| [0008](0008-postgres-fts.md) | Postgres full-text search, not a search engine | Proposed |
| [0009](0009-service-to-service-auth.md) | OpenBao short-lived credentials now, SPIFFE/mTLS later | Proposed |
| [0010](0010-pwa-not-native.md) | Installed PWA, no native codebase | Proposed — conditional on hardware answers |
| [0011](0011-podman-prod-docker-local.md) | Podman quadlets in production, Docker locally | Proposed |
| [0012](0012-connect-grpc-web.md) | Connect for gRPC-Web, grpc-gateway for partner REST | Proposed |
| [0013](0013-sqlc-not-orm.md) | sqlc, plain SQL, forward-only migrations | Proposed |

## The three that carry the most weight

If you read only three: [0005](0005-event-sourced-writes.md) (why a sale cannot conflict),
[0003](0003-offline-scope.md) (where the offline line is drawn), and
[0006](0006-oversell-accepted.md) (the business decision the whole sync design rests on).

## Process

1. Copy [`../templates/adr.template.md`](../templates/adr.template.md) to
   `NNNN-kebab-title.md`, taking the next number. Numbers are never reused or renumbered.
2. Open a PR with status `Proposed`.
3. Needs one maintainer approval plus one from anyone materially affected.
4. **Any prerequisite the ADR itself records must be satisfied and the outcome written
   into the ADR before it can be merged as `Accepted`.** Generic approval is not
   sufficient. An ADR whose status carries a qualifier — business sign-off, a blocking
   question, a conditional dependency — stays `Proposed` until that qualifier is
   resolved in the text, with the decision-maker and date recorded. The index Status
   column must carry the same qualifier so an unresolved prerequisite is visible without
   opening the file.
5. Merged as `Accepted`.
6. **Never edit an accepted ADR** except to change its status. Write a superseding ADR and
   link both ways.
7. Never delete an ADR. A wrong decision with recorded reasoning is more useful than no
   record.

### When an ADR is required

- Choosing a framework, library, or major dependency
- Designing or materially changing a data model
- Anything affecting the offline guarantee or the sync protocol
- Authentication or authorisation strategy
- API protocol or contract strategy
- Infrastructure, hosting, or deployment model
- Anything overriding a constraint recorded in `docs/architecture/`

### Status lifecycle

```text
Proposed → Accepted → (Superseded by ADR-NNNN | Deprecated)
```
