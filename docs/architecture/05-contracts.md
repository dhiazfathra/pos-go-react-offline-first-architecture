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
    └── pricing/         golden corpus shared by Go and TS suites (§4.8)
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
  # Go
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/connectrpc/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/gateway     # REST from google.api.http
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/openapiv2   # feeds Scalar + Postman
    out: docs/api

  # TypeScript
  - remote: buf.build/bufbuild/es
    out: gen/ts
    opt: target=ts
  - remote: buf.build/connectrpc/es
    out: gen/ts
    opt: target=ts
```

Two things intentionally *not* generated:

- **Zod schemas** come from a small in-repo generator (`scripts/proto-to-zod.ts`) reading
  the `protovalidate` options, rather than from an off-the-shelf plugin. Third-party
  proto→Zod plugins cover the type shape but not the validation rules, which is the half
  that matters — the whole point is that a form rejects locally exactly what the server
  would reject after a two-day sync.
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
export * from '../../gen/ts/pos/sales/v1/sales_pb';
export * from './zod';        // generated from protovalidate rules
export * from './money';      // helpers over common.v1.Money
```

**Money is never a float.** `common.v1.Money` is `{ currency_code: string, units: int64,
nanos: int32 }`, mapped to a minor-unit integer type in both languages. A `float64` price
is the most common way a POS quietly loses cents, and it is worth encoding the prohibition
in the shared type rather than in a style guide.

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
- run: buf generate && git diff --exit-code gen/   # no drift, no hand-edits
```

The last line is what makes committed generated code safe. It must be fast and it must not
be skippable.

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
| Generate-and-diff | `gen/` matches the protos; nothing hand-edited |
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
4. **Money type.** `common.v1.Money` (units + nanos) versus a plain minor-unit integer.
   Nanos handle sub-cent pricing (fuel, weighed goods); the integer is simpler and harder
   to misuse. Depends on whether sub-cent unit pricing is in scope.
5. **Field-level PII annotations.** A custom proto option marking PII would let redaction
   in logs, exports and Loki be generated rather than remembered. Cheap to add now,
   expensive to retrofit across a live event store.
