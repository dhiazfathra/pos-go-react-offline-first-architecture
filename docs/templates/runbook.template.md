# Runbook: <operation>

> **Audience:** on-call engineer, possibly at 3am, possibly not the author.
> Write for someone competent but without context. No cleverness, no implicit steps.

## Status

`Draft` | `Ready` | `Verified` — see
[`../runbooks/README.md`](../runbooks/README.md#status) for what each means.

## Dependencies and blockers

What must exist or be decided before this runbook is trustworthy. One line each, with the
open question or ADR it depends on. `None` if the runbook is fully executable today.
Anything listed here is a production-launch blocker, not documentation debt — mark the
affected steps ⚠️ in place as well.

## When to use this

The symptom or trigger. Be specific enough that someone can tell whether they are in the
right runbook within ten seconds.

## Severity and impact

What is broken for whom while this is happening. For POS specifically: **can stores still
take money?** If yes, this is not a store-down incident even if the server is unavailable —
say so explicitly, because it changes the urgency.

## Prerequisites

- Access required (which systems, which role)
- Tools required
- Anything that must be true before starting

## Steps

1. **Verify the symptom.** The exact command, and the exact output that confirms it.

   ```bash
   ```

2. **Assess blast radius.** How many tenants, stores, devices.
3. **Mitigate.** Stop the bleeding before diagnosing.
4. **Diagnose.**
5. **Resolve.**
6. **Verify resolution.** The command and the output that proves it is fixed. Not "check it
   works".

Each step: what to run, what you expect to see, and what to do if you see something else.

## Rollback

If the steps above make things worse, how to get back. If there is no way back, say so at
the top of the runbook, not here.

## Escalation

| Condition | Escalate to |
|---|---|
| … | … |

## Verification

How to confirm the system is healthy. Specific dashboards, specific queries, specific
expected values.

## Post-incident

- What to record
- Whether a postmortem is required
- Whether this runbook needs updating — if you deviated from it, it does
