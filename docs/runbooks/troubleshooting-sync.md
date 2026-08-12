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

`STATUS_REJECTED` in `EventResult` ([02-server §2.4](../architecture/02-server.md#event-storage))
is permanent by definition — quarantined, never auto-retried. An expired token or a
transient infrastructure failure is a transport-layer condition, not a rejection: it
never reaches `event_rejections` in the first place, because the device sees a
connection failure and retries with backoff ([§4.5](../architecture/04-sync.md#45-sync-protocol)),
the same as any other dropped request. If one of those shows up as a `reason` in this
table, that is itself the defect to chase — a transient failure is being misclassified as
permanent server-side.

Classify each genuine rejection before calling it a defect:

| Class | Example | Handling |
|---|---|---|
| Business exception | The device acted on a stale cache — item deleted, price changed, staff permission revoked since the last pull | Not a rejection at all: the server accepts the event, records the discrepancy as a reconciliation exception, and raises it to a manager. The sale happened; do not discard it |
| Terminal | Wrong store, malformed payload, schema violation, `EVENT_ID_REUSE` (same ID, different bytes), `SEQUENCE_REUSE` (`device_seq` collision) | Manager task on the device; needs a human or a fix before the event can ever be accepted |
| Business-rule violation | `online_required` category sold offline ([ADR-0006](../decisions/0006-oversell-accepted.md)) | The one named business-rule case that **is** `REJECTED` — quarantined and raised to a manager, because the client-side block that should have prevented it already failed |
| Defect | A business rule the device evaluated correctly is rejected server-side, for a reason not listed above | **Server bug.** Fix the server and have the device re-push |

`STATUS_DEFERRED` is not in this table at all — it belongs to genuine dependency ordering
(§4.5) and is retried automatically, never quarantined, never a manager task.

Only the last class is a server bug. A business-rule rejection means the server is
rejecting something that already physically happened — decide first whether that is stale
device state (business exception, accepted), the one contract-defined violation
(`online_required`, rejected by design), or genuinely divergent rule evaluation (defect),
because the remedies are completely different.

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
