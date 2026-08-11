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

`--dry-run` computes and reports differences without writing. It prints a **rebuild ID**
that captures this scope (projection, tenant, store, from) plus the starting event-log
watermark. Every later step takes `--rebuild <id>` and inherits that scope — the CLI must
not accept an unscoped `promote`, because a promote scoped more widely than the rebuild
would overwrite rows that were never recomputed.

### 2. Review the diff

Expect changes only where the defect applied. Unexpected differences elsewhere mean the
new projection code has a second behaviour change — stop and investigate.

### 3. Rebuild into a shadow table

```bash
pos-projections rebuild --rebuild <id> --shadow
```

Rebuilding in place leaves a window where reports read half-rebuilt data. A shadow table
plus an atomic swap avoids that entirely, and gives a trivially fast way back.

The swap is atomic; the *rebuild* is not. Live ingest keeps appending events while the
shadow table is being built, so the shadow is stale the moment the scan finishes. The
sequence that makes promotion safe is:

1. Record the server-side event-log watermark before the scan starts (the rebuild ID holds
   it).
2. Build the shadow table up to that watermark.
3. Catch up: apply every event appended after the watermark, repeating until the remaining
   backlog is zero.
4. Promote only while the backlog is zero, inside the same transaction as the swap, so no
   event can slip in between the check and the rename.

Promoting on the strength of the atomic swap alone silently drops every event that arrived
during the rebuild.

### 4. Compare

```bash
pos-projections compare --rebuild <id>
```

### 5. Swap

```bash
pos-projections promote --rebuild <id>
```

Atomic rename inside a transaction. `promote` re-checks the watermark backlog and refuses
to run if it is non-zero.

### 6. Verify

- Spot-check known-good figures against manually computed values from the event log
- Confirm shift and daily totals reconcile
- Confirm the reporting API returns the corrected numbers

## Rollback

The previous projection table is retained as `<name>_prev`. It is frozen at the moment of
the promote, so it is missing everything ingested since — swapping straight back loses those
events. Catch it up to the current watermark first, then promote:

```bash
pos-projections catchup --rebuild <id> --from-prev
pos-projections promote --rebuild <id> --from-prev
```

## Notes

- `occurred_at` is never used for ordering. Device clocks are untrusted
  ([§2.4](../architecture/02-server.md#event-storage)); ordering by it would let a device
  with a wrong clock reorder history.
- Rebuilds need a **total, deterministic** replay order, and `received_at` alone does not
  provide one: it is a timestamp with no uniqueness constraint and no tie-breaker, so two
  events written in the same instant can replay in either order and two rebuilds of the same
  log can disagree. Replay must instead use an immutable server-assigned append sequence (or
  persisted batch position) on `domain_events`, which is also what the rebuild watermark
  refers to. ⚠️ **That column does not exist in the schema yet** — it is a prerequisite for
  this runbook, not an optimisation.
- Large rebuilds should be batched and rate-limited to avoid competing with live ingest.
  **Ingest always wins** — a rebuild is never a reason for `PushEvents` to slow down.
- Rebuild correctness is covered by an automated test that rebuilds a fixture log and
  asserts the result. If that test does not exist yet, it is a higher priority than this
  runbook.

## Escalation

If a rebuild produces figures that disagree with counted cash for a closed period,
escalate to the backend lead and a business stakeholder before promoting. Changing
historical financial figures is not an engineering-only decision.
