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
| Application bug, additive migration (new column/table) | ✅ Redeploy previous image — the old code ignores the new column |
| Migration itself is the problem | ❌ Forward-fix or restore |
| Destructive migration already applied | ❌ Restore from PITR |

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

⚠️ **Recovery requires a device-side re-push mechanism for a given sequence range.** This
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
