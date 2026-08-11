# Runbook: Sync troubleshooting

> **Draft.** Queries are illustrative against the schema in
> `docs/architecture/02-server.md`.

## When to use this

- A device is not syncing
- Outbox depth or age alert fired
- Quarantined events exist
- A `device_seq` gap was detected
- A store reports missing sales in reports

## Severity ladder

| Symptom | Severity | Why |
|---|---|---|
| `device_seq` gap | **Critical** | Data loss. Sales exist that the server never received. |
| Quarantined events | High | Real sales are not in the ledger |
| Outbox age > 4 h in trading hours | High | Device may be dark; risk grows with time |
| Outbox depth > 500 | Medium | Backlog draining slowly, or device offline |
| Pull cursor lag | Low | Stale reads, no data at risk |

**Sales sitting in a device outbox are not lost.** They are undelivered. The urgency is
that the longer they sit, the more exposure there is to device loss, damage or wipe.

## Diagnosis

### 1. Which devices, and since when?

```sql
SELECT device_id, store_id, max(received_at) AS last_seen, max(device_seq) AS last_seq
FROM domain_events
WHERE tenant_id = $1
GROUP BY device_id, store_id
ORDER BY last_seen;
```

A device absent from recent rows is either offline, powered off, or failing to push.
Distinguishing these requires contacting the store — there is no server-side signal that
separates "no network" from "tablet in a drawer".

### 2. Sequence gaps

```sql
SELECT device_id, device_seq,
       device_seq - lag(device_seq) OVER (PARTITION BY device_id ORDER BY device_seq) AS gap
FROM domain_events
WHERE tenant_id = $1 AND device_id = $2
ORDER BY device_seq;
```

Any `gap > 1` means the server is missing sequence numbers that the device allocated.
**Treat it as data loss and escalate immediately** — but confirm before declaring it
permanent. This query only sees the server side.

Reconcile against the device before concluding anything:

| Where the missing sequence is | What it means | Action |
|---|---|---|
| Still pending in the device outbox | Undelivered, not lost | Get the device online; it drains itself |
| In the device's quarantine queue | Rejected, recoverable | Diagnose the rejection reason (§3), fix, re-push |
| Device has it marked drained, server does not have it | **Real loss** | Full escalation; recovery needs `RepushRange` |
| Device never allocated it (its own range skips it) | Client sequence-allocation defect | Backend + frontend; not a loss of a real sale |

Causes, in order of likelihood: device storage eviction, app data cleared, device wiped or
replaced without draining, or a client bug in sequence allocation.

### 3. Quarantined events

Events rejected by the server sit in the device's quarantine queue and surface as a manager
task. Server-side, look at rejection reasons:

```sql
SELECT reason, count(*) FROM event_rejections
WHERE tenant_id = $1 AND rejected_at > now() - interval '24 hours'
GROUP BY reason ORDER BY 2 DESC;
```

Rejection should be rare by design — auth failure, schema violation, or wrong store only
([§4.5](../architecture/04-sync.md#45-sync-protocol)). Classify each rejection before
calling it a defect:

| Class | Example | Handling |
|---|---|---|
| Retryable | Expired token, transient auth or infrastructure failure | Device retries; no intervention beyond restoring the dependency |
| Reconciliation | The device acted on a stale cache — item deleted, price changed, staff permission revoked since the last pull | The sale happened. Accept it and record the discrepancy as a business exception; do not silently discard |
| Terminal | Wrong store, malformed payload, schema violation | Manager task on the device; needs a human or a fix before the event can ever be accepted |
| Defect | A business rule the device evaluated correctly is rejected server-side | **Server bug.** Fix the server and have the device re-push |

Only the last class is a server bug. A business-rule rejection means the server is
rejecting something that already physically happened — decide first whether that is stale
device state (reconciliation) or genuinely divergent rule evaluation (defect), because the
remedies are completely different.

### 4. Pricing mismatches

```sql
SELECT count(*) FROM sales
WHERE tenant_id = $1 AND client_total <> server_total
  AND received_at > now() - interval '24 hours';
```

Non-zero does **not** immediately mean the Go and TypeScript pricing implementations have
diverged. The far more common cause is different *inputs*: the device priced against a
catalog, price list or promotion version it had pulled at the time, and the server recomputed
against the current one. Rule out inputs first.

1. Compare the recorded pricing inputs on the event — catalog/price-list version, promotion
   set, tax table — against what the server used.
2. If the versions differ, this is stale device state, not drift. Expected, and handled as a
   business exception below.
3. Only when the inputs and their versions match on both sides is this genuine
   implementation drift
   ([§4.8](../architecture/04-sync.md#48-the-duplicated-domain-logic-problem)). Each such row
   has a dollar value. Find the divergent rule, add the case to the golden corpus in
   `api/testdata/pricing/`, fix both implementations.

The customer paid what the device displayed. Do not "correct" historical sales; record the
discrepancy and handle it as a business exception.

## Mitigation

| Cause | Action |
|---|---|
| Device offline (network) | Contact store; confirm connectivity; device drains automatically |
| Device offline (powered off / in a drawer) | Contact store; power on and leave online until drained |
| Rate limited | Check limits — `PushEvents` limits should be generous; a backlog draining is legitimate traffic and must not be throttled into permanence |
| Server rejecting valid events | Server bug. Fix, deploy, device re-pushes. |
| Storage eviction on device | Check quota telemetry; assess data loss extent; escalate |
| Large backlog draining slowly | Verify batch size and that the till stays usable while draining |

## Escalation

| Condition | Escalate to |
|---|---|
| Any `device_seq` gap | Immediate — backend lead + business stakeholder |
| Quarantined events from a server-side bug | Backend on-call |
| Pricing mismatch with matching input versions | Backend + frontend — cross-implementation defect |
| Device outbox age > 24 h | Store operations, and consider device retrieval |

## Verification

- Device's `last_seen` is current
- Outbox depth zero
- No sequence gaps
- Shift totals reconcile against counted cash

## Post-incident

Any `device_seq` gap requires a postmortem regardless of how few events were lost. Loss of
financial records is the failure mode this architecture exists to prevent, and one instance
means an assumption is wrong.
