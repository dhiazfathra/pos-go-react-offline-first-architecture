# Runbook: Rollback

> **Draft.** Commands to be finalised once infrastructure exists.

## When to use this

A deploy has made things worse and forward-fixing is slower than reverting.

## The thing to understand first

**Migrations are forward-only.** There are no down-migrations in production
([ADR-0013](../decisions/0013-sqlc-not-orm.md)). This means:

| Situation | Rollback available? |
|---|---|
| Application bug, no migration in this release | ✅ Redeploy previous image |
| Application bug, additive migration (new column/table) | ⚠️ Only after a backward-compatibility check passes — see below |
| Migration itself is the problem | ❌ Forward-fix or restore |
| Destructive migration already applied | ❌ Restore from PITR |

"Additive" is not the same as "backward-compatible". A new column the old code ignores is
safe; a `NOT NULL` column without a default, a new constraint or trigger, an enum value the
old binary cannot parse, or a changed default all break the previous image on write. Confirm
the previous image runs against the *migrated* schema before treating rollback as available.

This is why migrations must be backward-compatible with the running version, and why
destructive changes are split across three releases. The discipline in the deploy path is
what makes rollback possible at all.

## Severity and impact

Stores keep trading. Sync is delayed, not lost — devices hold their outbox. You have more
time than instinct suggests. Use it to decide correctly rather than quickly.

## Steps

### 1. Decide: roll back, or forward-fix?

Roll back if the previous version is known good and no incompatible migration ran.
Forward-fix if the migration is the problem, or if rolling back would lose data written by
the new version.

Two compatibility gates, both of which must pass:

1. **Schema.** The previous image must run correctly against the already-migrated schema
   (see the table above).
2. **Events.** The previous image must be able to decode every event schema version that
   active devices can still push. Devices hold durable Protobuf events in their outbox, so
   an event type or field introduced by the new release will arrive at the old binary.

If the event gate cannot be confirmed, do not roll back the image: forward-fix, or accept
the affected events into the quarantine queue and replay them once a compatible version is
running. Migration compatibility alone is not sufficient.

### 2. Roll back the application

```bash
# redeploy the previous image SHA
systemctl --user set-environment POS_IMAGE_SHA=<previous-sha>
systemctl --user restart pos-server.service
```

### 3. Verify

Same checks as [`deploy.md` §4](deploy.md#4-verify). Confirm `PushEvents` recovers first.

### 4. If a destructive migration ran

Restore from point-in-time recovery. This is the only path, and it loses data written
since the restore point.

⚠️ **PITR restore procedure and RPO/RTO targets are undefined** — see
[`disaster-recovery.md`](disaster-recovery.md) and §6 Open Question 5. This must be
written and rehearsed before production launch. Discovering it during an incident is not
acceptable for a system holding financial records.

### 5. Reconcile ingested events

If a restore occurred, some acknowledged events may be lost from the server while devices
have already marked them drained. This is the worst case in the system.

Detection: `device_seq` gaps ([§4.10](../architecture/04-sync.md#410-observability)).

A gap is a *suspicion* of loss, not proof of it. Before declaring data loss, reconcile
against the device: compare the device-reported sequence range with what the device still
holds in its outbox and quarantine queue. A gap can equally mean the event was never
delivered (still pending on the device — it will drain on its own) or that sequence
allocation itself is defective. Only a sequence the device has already marked drained, and
which the server does not have, is real loss.

⚠️ **Recovery of a drained-but-missing event requires a device-side re-push mechanism for a
given sequence range (`RepushRange`).** This
does not exist yet and is a **required feature before production launch** — without it, a
restore permanently loses sales that devices believe were delivered. Raise this as a build
item, not a runbook gap.

## Escalation

| Condition | Escalate to |
|---|---|
| Restore required | Backend on-call + platform lead + business stakeholder |
| `device_seq` gaps after restore | Immediate, all hands — data loss |
| Rollback does not restore service | Do not repeat it; escalate |

## Post-incident

Postmortem required for any rollback that reached production. Specifically record whether
the migration discipline (backward compatibility, expand/migrate/contract) was followed,
because that is what determines whether the next rollback is possible.
