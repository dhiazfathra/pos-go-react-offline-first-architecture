# ADR-0010: Installed PWA, no separate native codebase

## Status

Proposed — **conditional on hardware access questions being answered**

## Date

2026-08-11

## Context

The POS must run on till tablets, staff phones and back-office desktops. The stack
constraint specifies a PWA reusing the frontend stack, with no separate native codebase.

For most of the requirement this is straightforward: one React codebase, one design
system, one test suite, instant updates without app-store review, and no second team.

The difficulty is hardware. A POS is not a document app — it talks to peripherals:

| Peripheral | Web API | Availability |
|---|---|---|
| Barcode scanner | Keyboard emulation | Universal — scanners present as keyboards |
| Thermal receipt printer | Web Serial / Web USB / Web Bluetooth | Chromium desktop and Android. **Absent on iOS Safari.** |
| Cash drawer | Usually via the printer | Follows the printer |
| Card terminal | Vendor SDK, or network/cloud | Highly vendor-specific |
| Customer display | Second window / HDMI | Workable |

There is also the storage question. Browsers may evict IndexedDB under pressure, and the
outbox is the one structure whose loss means lost sales
([ADR-0004](0004-replication-tiers.md)).

## Decision

Ship an installed PWA as the only client, with explicit conditions and a defined fallback.

- Installed via the browser's install prompt; service worker precaches all route chunks so
  a cold start works with no network.
- `navigator.storage.persist()` requested at install. Installed PWAs are normally granted
  persistence on the target platforms, which removes automatic eviction under quota
  pressure as a routine risk.
- Storage quota is monitored and surfaced before it becomes critical; Tier 2 history is
  evicted first, the outbox never.
- **Persistence is requested, not guaranteed, and it protects against less than it sounds
  like.** `persist()` can return `false`, in which case the origin's storage remains subject
  to automatic eviction. Even when granted, it does not survive the user clearing site data,
  the browser being reset, or the device being wiped or reimaged. The outbox is the only
  copy of an unsynced sale, so the claim "the outbox is never evicted" is a policy about our
  own eviction order, not a durability guarantee from the platform. Three mitigations follow
  and are part of this decision, not follow-ups:
  1. The result of `persist()` is checked, not fired and forgotten. If persistence is denied,
     the device shows a persistent warning and is flagged in the fleet view; whether that
     also gates trading is a business call for the same sign-off as
     [ADR-0006](0006-oversell-accepted.md).
  2. The outbox drains eagerly — on every connectivity regain and on a short timer — so the
     window in which the browser holds the only copy is minutes, not a shift.
  3. An operator-accessible export of the pending outbox exists for the "device is being
     reimaged" and "device is failing" cases, along with a documented recovery path in the
     runbooks. Devices are MDM-managed so that site-data clearing is not a thing a cashier
     can do casually. The export is the raw `pending`/`inflight`/`drained` outbox rows as
     a signed newline-delimited Protobuf file (same wire bytes the server validates, so
     nothing is re-encoded); import re-queues every entry to `pending` on the receiving
     device and relies on the same `eventId` idempotency as any other push to make a
     double-import harmless. The transfer itself is operator-to-operator (USB/local
     network under MDM control), not a new server endpoint.
- If a deployment genuinely requires zero lost sales under device loss, the browser outbox
  alone does not deliver it and the native-shell fallback below becomes mandatory rather
  than conditional.
- **Hardware peripherals are accessed via Web Serial / Web USB / Web Bluetooth**, which
  constrains the supported device platform for tills to Chromium-based browsers.
- Phones and back-office use, which need no peripherals, are unconstrained.

**This decision is conditional.** If iOS tablets with thermal printers are a hard
requirement, the PWA-only decision fails, because the required APIs do not exist on iOS
Safari and no amount of engineering works around that.

**Fallback, if triggered:** a thin native shell (Tauri, or Capacitor) wrapping the *same*
web application, adding only peripheral bridges and durable storage. This preserves the
single codebase — it is a shell, not a second client. It should not be built pre-emptively;
building it before it is needed means maintaining a native build pipeline for a capability
nobody has requested.

## Alternatives Considered

### React Native / native iOS + Android

- Pros: full hardware access, native storage guarantees, app-store distribution.
- Cons: a second codebase, second test suite, second release process; app-store review in
  the path of a hotfix — unacceptable when a pricing bug is live in stores.
- Rejected: the cost is very large and the benefit is limited to peripherals.

### Electron desktop app

- Pros: full hardware access, guaranteed storage, mature.
- Cons: desktop only, which does not serve tablet tills; large binary; separate update
  channel.
- Rejected as the primary client. Note that Tauri covers this ground more cheaply if a
  shell is later needed.

### Tauri shell from the start

- Pros: same web codebase, hardware access, real filesystem storage, small binary.
- Cons: a native build and signing pipeline per platform, and app distribution — real
  ongoing cost for a capability that may not be required.
- Rejected as the default, held as the defined fallback.

### PWA plus a small local companion agent

- Pros: PWA everywhere; the agent handles peripherals over localhost.
- Cons: a second thing to install and update on every till; a localhost service is its own
  security surface.
- Rejected unless the fallback is triggered on a platform where a shell is not viable.

## Consequences

- One codebase, one design system, one test suite, one release process.
- Updates ship instantly with no app-store review — critical when a pricing defect is live.
- **Till devices are effectively constrained to Chromium-based browsers**, which is a
  procurement constraint and must be stated in the README and to customers, not discovered.
- Storage persistence is granted rather than guaranteed. Monitored, mitigated, and an
  accepted residual risk — and the most likely single reason this ADR gets superseded.
- Peripheral integration is per-model work regardless of platform; the web APIs make it
  somewhat harder, not qualitatively different.
- If the fallback triggers, the change is a shell around the existing app, not a rewrite.
  Keeping peripheral access behind a narrow interface in `packages/` from day one is what
  makes that true, and is worth doing now.

## Blocking questions

1. Are iOS tablets a requirement for tills? If yes, this ADR fails as written.
2. Which receipt printers, and do they support Web Serial/USB/Bluetooth?
3. Which card terminals, and do they offer a network/cloud API rather than a native SDK?

These must be answered before this ADR moves from Proposed to Accepted.

## Related

- [ADR-0004](0004-replication-tiers.md)
- [Architecture §3](../architecture/03-client.md)
