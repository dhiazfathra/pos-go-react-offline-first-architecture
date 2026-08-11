# 5. Shared Contracts and Schemas

## 5.1 Where they live

`api/` at the repo root — a sibling of `apps/`, not a child of either.

```
buf.yaml  buf.gen.yaml  buf.lock    # at the repo root, alongside api/
api/
├── pos/
│   ├── common/v1/       money.proto, ids.proto, page.proto, error.proto
│   ├── identity/v1/
│   ├── catalog/v1/
│   ├── pricing/v1/
│   ├── inventory/v1/
│   ├── sales/v1/
│   ├── shifts/v1/       shift.proto (ShiftOpened, CashMoved)
│   ├── customers/v1/    customer.proto (CustomerChanged)
│   ├── reporting/v1/
│   └── sync/v1/         envelope.proto, push.proto, pull.proto, policy.proto
└── testdata/
    ├── pricing/         golden corpus shared by Go and TS suites (§4.8)
    ├── money/           rounding and conversion vectors (§5.4)
    └── validation/      Protovalidate ⇄ Zod agreement fixtures (§5.3)
```

The buf config files live at the repo root, not inside `api/`, so the `out:` paths in
§5.3 (`gen/go`, `gen/ts`, `docs/api`) resolve against the root — see the layout in
[§1.5](01-overview.md#15-monorepo-layout).

Neither language owns the contract. Putting protos under `apps/server` would make the
server the de-facto owner and invite server-shaped API design — request/response types
that mirror internal Go structs rather than what a client needs. The physical location is
doing real work here.

`api/testdata/` sits with the contract for the same reason: the pricing golden corpus is
part of the agreement between client and server, not a test fixture belonging to either.

## 5.2 What Protobuf is the source of truth for

Everything crossing a boundary:

| Contract | Location | Consumers |
|---|---|---|
| gRPC + REST API | `pos/*/v1/*.proto` | Go server, TS client, partners |
| Offline event payloads | `pos/sync/v1/envelope.proto` + domain events | Client outbox, server ingest, event table |
| Validation rules | `protovalidate` options inline | Go middleware, generated Zod |
| Pricing rule definitions | `pos/pricing/v1/rules.proto` | Both pricing implementations |
| Replication policy | `pos/sync/v1/policy.proto` | Server policy, client sync engine |

The second row is the one that is easy to miss and expensive to get wrong. **Events stored
in Postgres are serialised with the same schema used on the wire.** That means Buf's
breaking-change detection protects not only live API compatibility but the readability of
years of historical events. An event written by a client in 2026 must be decodable by a
server in 2033. Protobuf's wire-compatibility rules are precisely designed for this, which
is a large part of why they beat JSONB here.

## 5.3 Codegen

`buf generate` writes to `gen/`, which is committed and never hand-edited.

```yaml
# buf.gen.yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/dhiazfathra/pos/gen/go
plugins:
  # Go — every remote plugin is version-pinned; see below
  - remote: buf.build/protocolbuffers/go:v1.36.5
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go:v1.18.1
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/gateway:v2.26.1   # REST from google.api.http
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/openapiv2:v2.26.1 # feeds Scalar + Postman
    out: docs/api

  # TypeScript
  - remote: buf.build/bufbuild/es:v2.2.3
    out: gen/ts
    opt: target=ts
  - remote: buf.build/connectrpc/es:v2.0.0
    out: gen/ts
    opt: target=ts
```

(Versions above are illustrative — the rule is that a version is present, not which one.)

**Every remote plugin carries an explicit version, and so does the toolchain around it.**
An unversioned `remote:` resolves to whatever the BSR currently calls latest, which means
`buf generate` is not a pure function of the repo: the same commit generates different
output next month. With `gen/` committed and a generate-and-diff gate in CI (§5.5), that
turns into a build that breaks on an unrelated PR, at a time nobody chose, from a change
nobody reviewed. Pinning makes an upgrade a deliberate commit with a reviewable diff of
generated code — which is the main thing committing `gen/` buys.

The same applies to everything else that shapes generated output: the Buf CLI version is
pinned in CI and in the local toolchain manifest, and `scripts/proto-to-zod.ts` runs under
a lockfile-pinned dependency set. A pinned plugin behind an unpinned CLI is still a
floating build.

Two things intentionally *not* generated:

- **Zod schemas** come from a small in-repo generator (`scripts/proto-to-zod.ts`) reading
  the `protovalidate` options, rather than from an off-the-shelf plugin. Third-party
  proto→Zod plugins cover the type shape but not the validation rules, which is the half
  that matters — the whole point is that a form rejects locally exactly what the server
  would reject after a two-day sync.

  **The generator fails closed.** Protovalidate rules are CEL, and CEL is more expressive
  than anything that can be mechanically lowered into Zod — message-level rules,
  cross-field predicates, and arbitrary custom expressions among them. When the generator
  meets a rule it cannot faithfully translate, it **exits non-zero and names the field**.
  It never emits a schema with the rule silently omitted. Failing open is the worse
  outcome by a wide margin: a form that quietly stops enforcing a rule looks like it works
  and produces events that pass locally, sit in an outbox for two days, and are `REJECTED`
  on arrival — the exact failure this generator exists to prevent, made harder to spot
  because CI was green. When a rule genuinely cannot be lowered, the escape hatch is an
  explicit per-field opt-out in the generator config, so the gap is a reviewed line in a
  file rather than an absence nobody can see.

  Agreement is verified, not assumed: a shared fixture corpus of valid and invalid
  messages — covering field-level, message-level and cross-field rules — is run through Go
  Protovalidate and the generated Zod schemas, and the two must return the same verdict
  for every case. A divergence fails both CI jobs, the same arrangement the pricing corpus
  uses (§4.8).
- **Domain types.** Generated types are wire types. The server maps them to `biz` domain
  types at the `service` layer; the client maps them at the feature boundary. Using
  generated types as domain types couples the domain to the wire format, and every wire
  change becomes a domain change.

## 5.4 Consumption

**Go** — `gen/go` is a module in `go.work`. `service` packages import it; `biz` packages
are blocked from importing it by `depguard`.

**TypeScript** — `packages/contracts` re-exports `gen/ts` plus the derived Zod schemas.
Application code imports `@pos/contracts`, never `gen/ts` directly, so the indirection
gives one place to add branded types, helpers, and the occasional compatibility shim.

```ts
// packages/contracts/src/index.ts
export * from '../../../gen/ts/pos/sales/v1/sales_pb';   // gen/ is at the repo root
export * from './zod';        // generated from protovalidate rules
export * from './money';      // helpers over common.v1.Money
```

**Money is never a float.** A `float64` price is the most common way a POS quietly loses
cents, and it is worth encoding the prohibition in the shared type rather than in a style
guide.

`common.v1.Money` is `{ currency_code: string, units: int64, nanos: int32 }` — the
canonical wire representation, chosen because it survives sub-cent unit pricing (fuel,
weighed goods, per-gram tariffs) that a minor-unit integer cannot express at all.

The in-language mapping is **not** a plain minor-unit integer, and saying it was would be
a real bug rather than a wording slip: nanos give nine decimal places, a minor-unit
integer gives two, and any lossy conversion done independently in Go and TypeScript is a
licence for the two to round differently and disagree on a total. Instead:

| Layer | Representation | Rule |
|---|---|---|
| Wire and event payloads | `Money` (`units` + `nanos`) | Canonical. `nanos` and `units` must share a sign; `\|nanos\| < 1e9` |
| Arithmetic, both languages | Exact scaled integer — total nanos as `int64` (Go) / `bigint` (TS) | All pricing maths happens here. No float, ever, at any step |
| Presentation and tender | Minor units | The **only** place rounding occurs |

One rounding rule, applied at one point. A basket is priced end to end in nanos with no
intermediate rounding, and the result is rounded to the currency's minor unit exactly once
— at the point a total is displayed, charged, or written as a tender amount — using
half-up on the absolute value (so −0.5 rounds away from zero, symmetrically). Rounding per
line item instead of once per total produces totals that differ by a cent from what a
customer computes by hand, and rounding twice produces totals that differ between the
device and the server, which surfaces as a §4.8 pricing mismatch and burns a manager's
afternoon.

Currency minor-unit exponents come from a single shared table keyed by ISO 4217, not from
an assumed two decimal places; JPY has zero and KWD has three. Arithmetic across differing
`currency_code` values is an error, not a coercion.

Because this is precisely the kind of rule two implementations drift on, it is pinned by
shared vectors in `api/testdata/money/` — half-way cases, negatives, zero- and
three-decimal currencies, `int64` boundaries, and nanos that do not divide evenly into a
minor unit — run by both the Go and Vitest suites alongside the pricing corpus (§4.8).

## 5.5 Versioning and breaking changes

### Package versioning

Version lives in the package path: `pos.sales.v1`. A breaking change means `v2`, with both
served in parallel during migration. This is the only versioning scheme that works when
old clients cannot be forced to upgrade — and in a POS, a till that has been offline for a
week *is* an old client.

### CI enforcement

```yaml
- run: buf lint
- run: buf breaking --against 'https://github.com/…git#branch=main'
- run: buf format --diff --exit-code

# Run EVERY generator, then diff EVERY committed output path.
- run: buf generate                      # → gen/go, gen/ts, docs/api
- run: pnpm tsx scripts/proto-to-zod.ts  # → packages/contracts/src/zod
- run: git diff --exit-code -- gen/ docs/api packages/contracts/src/zod
- run: git status --porcelain --untracked-files=all -- gen/ docs/api \
         packages/contracts/src/zod | (! grep .)   # catches NEW generated files
```

This is what makes committed generated code safe, and the scope of the check is the whole
point. Diffing only `gen/` leaves two committed artifacts unguarded: the OpenAPI documents
`buf.gen.yaml` writes to `docs/api`, and the Zod schemas produced by the in-repo
generator. Both can go stale while CI stays green — and a stale Zod schema is the worst of
the three, because it means the client stops rejecting exactly what the server will
reject, which is the entire justification for generating it (§5.3). A drift check that
covers most outputs provides confidence proportional to all of them.

The `git diff` check alone is not sufficient: a newly generated file is untracked, and
`git diff` ignores untracked files, so adding an RPC and forgetting to commit its output
would pass. Hence the second assertion.

It must be fast and it must not be skippable.

### Rules

| Change | Allowed in `v1`? | Notes |
|---|---|---|
| Add a field | ✅ | New field numbers only |
| Add an RPC | ✅ | |
| Add an enum value | ⚠️ | Only if every consumer has an `UNSPECIFIED`/default branch |
| Rename a field | ❌ | Breaks JSON/REST consumers even though the wire is by number |
| Change a field type | ❌ | |
| Remove a field | ❌ | Deprecate, `reserved`, remove in `v2` |
| Change a field number | ❌ | Corrupts every stored historical event |
| Remove an RPC | ❌ | |

The last two are absolute because of the event store. A field number reused for a different
type does not just break the API — it silently misinterprets years of persisted events.
`reserved` on removed field numbers and names is mandatory, not advisory.

### The old-client problem

A device offline for a week returns with events serialised against last week's schema. The
server must accept them. Concretely:

- Buf breaking-change detection runs against `main`, so `v1` is only ever extended.
- Unknown fields are preserved, never dropped, on both sides.
- New required semantics are introduced as optional fields with a server-side default, and
  only become mandatory after a defined client-version floor is enforced.
- The server records the client version on every event, so "how many devices are still on
  the old schema" is a query rather than a guess.

**A breaking event-schema change is not deployable as a normal release.** It requires a
migration of stored events or a permanent decoder for the old version. Treat it as a
project, not a PR — and prefer never doing it.

## 5.6 API documentation

- **Scalar** serves the OpenAPI generated from the protos, at `/docs` in non-prod. Not
  hand-written, so it cannot drift.
- **Postman** collection generated from the same OpenAPI in CI and published as a build
  artifact. Hand-maintained Postman collections rot within weeks.
- **Buf Schema Registry** (or a self-hosted equivalent) publishes the module for partner
  consumption and gives partners generated SDKs without repo access.

## 5.7 Contract testing

| Test | What it proves |
|---|---|
| `buf lint` / `buf format` | Style consistency |
| `buf breaking` | No incompatible change reached `main` |
| Generate-and-diff | `gen/`, `docs/api` and the Zod output all match the protos; nothing hand-edited |
| Validation agreement | Go Protovalidate and generated Zod accept and reject the same messages — §5.3 |
| Money vectors | Go and TS agree on conversion and rounding — §5.4 |
| Round-trip | Go server ⇄ generated TS client over every RPC, in CI |
| Event decode compatibility | Events serialised by the previous release still decode |
| Pricing golden corpus | Go and TS pricing agree — §4.8 |

The event decode compatibility test is the unusual one and the one most worth having. It
keeps a fixture of serialised events from each released schema version and asserts the
current server decodes them all. It is the automated form of the promise that a device
offline since a previous release will not lose its sales.

---

## Tradeoffs

**Protobuf-first costs friction on every change.** Adding a field means editing a proto,
regenerating, and committing generated output before you can write the feature. Slower
than adding a key to a JSON response. Bought with it: no drift between Go and TypeScript,
enforced compatibility, and generated docs — which for a system whose defining risk is
old clients returning with old payloads is a very good trade.

**Committed `gen/`** — clean clone builds, reviewable contract diffs, IDE navigation.
Costs merge conflicts in generated files and the risk of a hand-edit that only the
generate-and-diff job catches. Keep that job fast and mandatory.

**Buf's breaking-change detection does not understand semantics.** Changing a field's
*meaning* while keeping its type passes every check and breaks every consumer. Only review
catches it. Worth calling out because green CI creates false confidence here.

**A custom proto→Zod generator is code we own.** A few hundred lines, and it must track
`protovalidate` as it evolves. Justified because the alternative — hand-written client
validation — reintroduces exactly the drift this whole section exists to prevent. Revisit
if a mature plugin covering validation rules appears.

**Failing closed will block work at inconvenient times.** Someone adds a message-level CEL
rule, the generator refuses, and a PR stops until either the generator learns the rule or
an opt-out is reviewed and merged. That friction is the feature: the alternative is a
silently weakened client schema whose cost lands two days later on a device that has been
offline, where it is far more expensive and far harder to attribute.

**Pinned plugin versions mean upgrades are manual.** Nobody gets fixes for free, and pins
go stale until someone bumps them. Accepted: a reproducible build is worth more than free
upgrades when the generated output is committed and gated. A scheduled dependency-update
PR keeps the staleness bounded and keeps each bump reviewable.

**Package-path versioning (`v1`/`v2`) is heavyweight** and encourages avoiding breaking
changes rather than managing them. Given the event store, avoidance is the correct
incentive.

**REST is generated, not designed.** The JSON surface is Protobuf's JSON mapping, which is
not always the shape a REST-native partner expects — `oneof` and well-known types map
awkwardly. Acceptable while REST is a secondary consumer. If a partner-facing REST API
becomes a product surface, it deserves a hand-designed façade rather than more annotations.

## Open Questions

1. **Buf Schema Registry: hosted or self-hosted?** Cost and vendor exposure versus
   operating it ourselves. Affects partner SDK distribution.
2. **Event schema evolution policy.** Concretely: what is the supported client-version
   floor, how long are old decoders kept, and what triggers forcing an upgrade? Needs a
   written policy before the second release, not the tenth.
3. **Do partners get gRPC, or REST only?** Decides how much the generated JSON surface
   needs to be curated.
4. **Money precision beyond the rounding rule.** The representation itself is settled in
   §5.4 — `Money` on the wire, exact scaled-integer arithmetic, one half-up rounding at
   presentation. What is still open is whether any target jurisdiction mandates a
   *different* rule: some tax regimes require per-line rounding, and some require
   banker's rounding on tax lines specifically. Either would change where the single
   rounding point sits. Needs a per-jurisdiction answer alongside the fiscal question in
   [§4](04-sync.md#open-questions) before the pricing engine is written twice.
5. **Field-level PII annotations.** A custom proto option marking PII would let redaction
   in logs, exports and Loki be generated rather than remembered. Cheap to add now,
   expensive to retrofit across a live event store.
