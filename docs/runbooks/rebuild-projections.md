# Runbook: Rebuild projections from the event log

> **Draft.** Commands illustrative until the CLI exists.

## When to use this

- A projection bug produced wrong figures
- After a restore, to guarantee projections match the event log
- After a projection schema change that alters how events are interpreted
- A report disagrees with the underlying events

## Why this is safe

Projections are **derived** data ([ADR-0005](../decisions/0005-event-sourced-writes.md)).
The event log is the source of truth. Rebuilding cannot lose information — it recomputes
from facts that are still there.

This is the operational payoff of event sourcing, and it is worth exercising deliberately
rather than only under pressure. If this runbook has never been run successfully, the
event log's main benefit is theoretical.

## Impact

Reports and any server-side read of the affected projection are stale or unavailable during
the rebuild. **Stores keep trading throughout** — devices do not read server projections
([ADR-0004](../decisions/0004-replication-tiers.md)); they read their local store.

## Prerequisites

- Confirm the defect is in the *projection*, not in the events themselves. If the events
  are wrong, rebuilding faithfully reproduces the wrong answer. Check a handful of raw
  events first:
  ```bash
  pos-events decode --event-id <uuid>
  ```
- Fix and deploy the projection code **before** rebuilding.
- Take a backup of the current projection tables for comparison.

## Steps

### 1. Scope the rebuild

Prefer the narrowest scope that fixes the problem.

```bash
pos-projections rebuild \
  --projection sales_daily \
  --tenant <id> \
  --store <id> \
  --from 2026-06-01 \
  --dry-run
```

`--dry-run` computes and reports differences without writing.

### 2. Review the diff

Expect changes only where the defect applied. Unexpected differences elsewhere mean the
new projection code has a second behaviour change — stop and investigate.

### 3. Rebuild into a shadow table

```bash
pos-projections rebuild --projection sales_daily --tenant <id> --shadow
```

Rebuilding in place leaves a window where reports read half-rebuilt data. A shadow table
plus an atomic swap avoids that entirely, and gives a trivially fast way back.

### 4. Compare

```bash
pos-projections compare --projection sales_daily --tenant <id>
```

### 5. Swap

```bash
pos-projections promote --projection sales_daily --tenant <id>
```

Atomic rename inside a transaction.

### 6. Verify

- Spot-check known-good figures against manually computed values from the event log
- Confirm shift and daily totals reconcile
- Confirm the reporting API returns the corrected numbers

## Rollback

The previous projection table is retained as `<name>_prev`. Swap back:

```bash
pos-projections promote --projection sales_daily --tenant <id> --from-prev
```

## Notes

- Rebuilds are ordered by `received_at`, not `occurred_at`. Device clocks are untrusted
  ([§2.4](../architecture/02-server.md#event-storage)); using `occurred_at` for ordering
  would let a device with a wrong clock reorder history.
- Large rebuilds should be batched and rate-limited to avoid competing with live ingest.
  **Ingest always wins** — a rebuild is never a reason for `PushEvents` to slow down.
- Rebuild correctness is covered by an automated test that rebuilds a fixture log and
  asserts the result. If that test does not exist yet, it is a higher priority than this
  runbook.

## Escalation

If a rebuild produces figures that disagree with counted cash for a closed period,
escalate to the backend lead and a business stakeholder before promoting. Changing
historical financial figures is not an engineering-only decision.
