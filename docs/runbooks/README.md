# Runbooks

Operational procedures. Written for an on-call engineer who may not have written the code.

| Runbook | Use when |
|---|---|
| [deploy.md](deploy.md) | Promoting a build to staging or production |
| [rollback.md](rollback.md) | A deploy made things worse |
| [troubleshooting-sync.md](troubleshooting-sync.md) | A device is not syncing, or a sync alert fired |
| [rebuild-projections.md](rebuild-projections.md) | Reports disagree with the event log |
| [disaster-recovery.md](disaster-recovery.md) | Data or infrastructure loss |

## Read this before your first incident

**Server down does not mean stores down.** Devices trade offline and queue writes locally.
This inverts the usual urgency: you generally have hours, not minutes, and the right move
is to decide correctly rather than quickly.

**The real clock is per-device outbox age.** Every hour a device holds undrained events is
exposure to permanent loss if that device is dropped, wiped, or stolen. The most likely
cause of permanent data loss in this system is physical, not infrastructural.

**Never tell a store to clear app data or factory-reset a device** to "fix" a sync error.
That is how a recoverable incident becomes permanent financial data loss. If a device is
misbehaving, get it online and let it drain first.

## Status

All runbooks are drafts pending infrastructure. Items marked ⚠️ depend on unresolved
decisions in [`../README.md`](../README.md#blocking-questions) and are blockers for
production launch, not documentation debt.

Template: [`../templates/runbook.template.md`](../templates/runbook.template.md).
