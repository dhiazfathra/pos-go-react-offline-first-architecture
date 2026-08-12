# Runbook: Disaster recovery

> **Draft — incomplete by design.** Several sections cannot be written until decisions in
> [§6 Open Questions](../architecture/06-platform.md#open-questions) are made. Gaps are
> marked ⚠️ and each is a blocker for production launch, not a documentation nicety.

## What "disaster" means here

Unusually for a web system, **server loss is not a store-down event.** Devices keep
trading offline ([ADR-0003](../decisions/0003-offline-scope.md)). This reframes the whole
DR posture:

| Scenario | Stores trading? | Real risk |
|---|---|---|
| Server unavailable | ✅ Yes, offline | Sync backlog grows; device-loss exposure grows |
| Database lost, restorable | ✅ Yes | Data loss between restore point and failure |
| Database lost, unrecoverable | ✅ Yes | Catastrophic — ledger gone |
| Region lost | ✅ Yes | As above, plus recovery time |
| Device lost or destroyed | ❌ That till | Any undrained events on it are gone permanently |

The last row is worth sitting with. **The most likely cause of permanent data loss in this
system is a dropped tablet, not a database failure.** Every hour a device holds undrained
events is exposure. This is why per-device outbox age is a first-class alert
([§4.10](../architecture/04-sync.md#410-observability)) and why "server is up" is the wrong
health metric.

## Recovery targets

⚠️ **Undefined.** Must be set before launch.

Proposed starting points, for discussion:

| Target | Proposed | Rationale |
|---|---|---|
| RPO (ingested data) | ≈ 0 | PITR with continuous WAL archiving. Financial records. |
| RTO (server) | 4 h | Stores trade offline meanwhile; the pressure is real but not immediate |
| RTO (read-only reporting) | 8 h | Lower priority than ingest |
| Max tolerable sync backlog | 24 h | Beyond this, device-loss exposure becomes unacceptable |

Note the asymmetry: RPO must be near zero, RTO can be hours. That is the opposite of a
typical web product and it should drive where recovery money is spent — WAL archiving and
restore rehearsal, not hot standby.

## Backup strategy

| Asset | Method | Frequency | Off-site copy | Retention | Tested |
|---|---|---|---|---|---|
| Postgres | PITR, continuous WAL + base backup | Continuous / daily base | WAL and base backups replicated to a second region, restorable without any resource in the primary region | ⚠️ TBD, statutory 7–10 y likely | ⚠️ Not yet |
| MinIO | Versioned bucket + offsite replication | Continuous | Cross-region bucket replication, separate credentials | ⚠️ TBD | ⚠️ Not yet |
| OpenBao | Snapshot | Daily | Snapshot replicated off-region; **unseal/recovery keys held separately from the snapshot**, split across custodians ⚠️ *unseal strategy undecided — ADR-0009* | 30 d | ⚠️ Not yet |
| IaC state | Remote state, versioned + encrypted | Per apply | State backend and its encryption key in a second region; state must be readable with no primary-region access | 90 d | ⚠️ Not yet |
| Container images | Registry, immutable by SHA | Per build | Registry replicated or mirrored off-region | 1 y | n/a |

Regional loss is in scope, so every control-plane dependency needs a copy that survives the
region and credentials that can reach it from outside the region. A backup stored in the
region it protects, or one whose decryption key is only obtainable from the failed region,
does not count.

**An untested backup is not a backup.** Restore rehearsal must be scheduled — quarterly at
minimum — and must be performed *from the off-site copy only*, with primary-region access
assumed unavailable. The rehearsal must include OpenBao key recovery and the
event-reconciliation step below, which is the part most likely to be wrong.

## Recovery procedure

### 1. Assess

- What is lost? Application only, or data?
- Are stores trading? (Almost certainly yes — confirm, and say so in the incident channel
  to set the right tempo.)
- Sync backlog size and oldest undrained event age.

### 2. Communicate

Tell stores explicitly: **keep trading, do not clear app data, do not factory-reset a
device, keep devices powered and online.** This message matters more than any technical
step — a well-meaning store manager clearing an app's storage to "fix the sync error" is a
realistic way to turn a recoverable incident into permanent loss.

### 3. Restore infrastructure

⚠️ Procedure TBD — depends on the deployment topology decision.

Ordering constraint: OpenBao must be restored and **unsealed** before the application can
obtain database credentials ([ADR-0009](../decisions/0009-service-to-service-auth.md)).
⚠️ Unseal strategy undecided. This is a hard blocker: without it, recovery cannot begin.

### 4. Restore Postgres to the latest consistent point

⚠️ Procedure TBD.

### 5. Reconcile devices — the step that matters

After a restore to a point before the failure, the server has lost events that devices
already saw acknowledged and marked drained.

Detect via `device_seq` gaps
([`troubleshooting-sync.md`](troubleshooting-sync.md#2-sequence-gaps)).

⚠️ **A device-side re-push mechanism for a given sequence range does not exist.** It is a
**required feature before production launch.** Without it, any restore permanently loses
sales that devices believe were delivered — which makes the PITR strategy only partly
effective and is the single most important gap in this document.

Requirements for that feature:
- Devices retain drained events for a retention window (proposed: 30 days) rather than
  deleting on ack.
- The server can request [`RepushRange(device_id, from_seq, to_seq)`](../architecture/04-sync.md#repushrange-the-device-repush-contract).
- Re-push is idempotent by `event_id`, so over-requesting is harmless.

Retaining drained events costs storage and is the correct trade: it converts a class of
permanent loss into a recoverable one.

### 6. Verify

- Event counts per device match device-reported sequences
- No `device_seq` gaps
- Projections rebuilt and consistent — see
  [`rebuild-projections.md`](rebuild-projections.md)
- Shift totals reconcile against counted cash for the affected period

### 7. Resume

Confirm all devices draining; monitor for 24 h.

## Escalation

Any data-loss scenario: backend lead, platform lead, and a business stakeholder
immediately. Financial data loss has legal and tax reporting consequences and is not an
engineering-only decision.

## Post-incident

Full postmortem, mandatory. Include: whether backups were current, whether the restore
rehearsal had been performed, whether device reconciliation worked, and how much financial
data was permanently lost.
