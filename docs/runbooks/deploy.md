# Runbook: Deploy

> **Draft.** Written against the architecture in `docs/architecture/06-platform.md`. Exact
> commands, hostnames and dashboard links must be filled in once infrastructure exists.
> Anything marked ⚠️ is a decision not yet made.

## When to use this

Promoting a build to staging or production.

## Severity and impact

Deploys are zero-downtime by design. **Stores keep trading during a deploy and during a
failed deploy** — devices queue writes locally ([ADR-0003](../decisions/0003-offline-scope.md)).
A failed deploy delays sync; it does not stop sales. Treat accordingly: do not rush a
rollback decision out of misplaced urgency.

## Prerequisites

- Image built in CI and tagged with the commit SHA
- All quality gates green ([§6.3](../architecture/06-platform.md#63-cicd))
- Staging soak completed (production only)
- OpenBao reachable and unsealed ⚠️ *unseal strategy undecided — see ADR-0009*

## Steps

### 1. Confirm what you are shipping

```bash
git log --oneline <current-sha>..<new-sha>
```

Check specifically for: migrations, proto changes, event-schema changes. Any of the three
changes the risk profile of this deploy.

### 2. Migrations

Migrations run as a separate job before the new binary starts.

```bash
# apply
systemctl --user start pos-migrate@<sha>.service
journalctl --user -u pos-migrate@<sha> -f
```

Migrations must be backward-compatible with the **currently running** version — the two
overlap. Anything destructive is expand → migrate → contract across three releases.

If a migration takes a lock on a large table, it can stall ingest. Verify against a
restored production-shaped dump in CI, not here.

### 3. Roll the application

```bash
systemctl --user restart pos-server.service
systemctl --user status pos-server.service
journalctl --user -u pos-server -f
```

⚠️ Multi-instance rolling strategy undecided — depends on the deployment topology open
question (§6). With a single instance there is a brief gap in which devices queue locally,
which is acceptable.

### 4. Verify

| Check | Expected |
|---|---|
| `/healthz` | 200 |
| `/readyz` | 200 |
| Version endpoint | new SHA |
| `PushEvents` success rate | returns to baseline within 2 min |
| Error rate (Sentry) | no new issue classes |
| Migration version | matches expected |

**The check that matters most:** `PushEvents` succeeding. If sales are being ingested, the
deploy is functionally good regardless of what else is noisy.

### 5. Watch

Fifteen minutes on the sync dashboard. Specifically per-device outbox age
([§4.10](../architecture/04-sync.md#410-observability)) — an aggregate number will not show
you the one store that has stopped syncing.

## Rollback

See [`rollback.md`](rollback.md). Short version: redeploy the previous image SHA. Do **not**
attempt to reverse migrations — they are forward-only.

## Escalation

| Condition | Escalate to |
|---|---|
| `PushEvents` failing > 5 min | Backend on-call |
| Migration stalled or holding locks | Backend on-call + DBA |
| Any device reporting a `device_seq` gap | Immediate — potential data loss |
| OpenBao sealed and unreachable | Platform on-call |

## Post-incident

If you deviated from this runbook, update it before you go to bed.
