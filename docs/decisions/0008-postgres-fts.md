# ADR-0008: Postgres full-text search, not a dedicated search engine

## Status
Proposed

## Date
2026-08-11

## Context

Two search surfaces exist, and they are different problems:

1. **Online search** — back-office product and customer lookup, reporting filters.
2. **Offline search at the till** — a cashier types three characters and needs the top ten
   matching products from their store's catalog, in under 50 ms, with no network.

The second is the high-frequency one, and it is served entirely client-side from the
Tier-1 in-memory index ([ADR-0004](0004-replication-tiers.md)). No server-side search
technology helps it at all.

So the decision is only about surface 1: server-side search over a store's catalog and
customer list — typically thousands to low tens of thousands of rows per tenant, always
scoped by `tenant_id` and usually by `store_id`.

## Decision

PostgreSQL `tsvector`/`tsquery` for server-side search.

- A generated `tsvector` column per searchable table, with a GIN index.
- `pg_trgm` for typo tolerance and partial matching on SKUs and barcodes — which matters
  far more at a till than semantic relevance does.
- Weighted vectors (`setweight`) so name beats description beats supplier notes.
- Always filtered by `tenant_id` first; the index is scoped, not global.

Client-side offline search uses a separate lightweight index over the in-memory Tier-1
data. The two implementations share a golden-file test corpus so they cannot drift into
ranking results differently — a cashier moving between online and offline will notice.

## Alternatives Considered

### Elasticsearch / OpenSearch
- Pros: superior relevance, faceting, aggregations, scales to very large corpora.
- Cons: a stateful service to operate, back up and upgrade; an indexing pipeline that is a
  new source of "search is stale" bugs; a second copy of the data with its own consistency
  story; substantial memory footprint.
- Rejected: no user-visible benefit at this corpus size, and it adds an availability
  dependency to a system whose whole premise is minimising dependencies.

### Meilisearch / Typesense
- Pros: much lighter than Elasticsearch, excellent typo tolerance, fast.
- Cons: still a separate stateful service and sync pipeline. Their strongest feature —
  instant typo-tolerant search-as-you-type — is exactly the feature we serve client-side
  from local data anyway.
- Rejected: solves the problem we already solved locally, and does not solve the one we
  have.

### `LIKE '%term%'`
- Pros: zero setup.
- Cons: no index usage, no ranking, no stemming; degrades badly.
- Rejected.

### Postgres FTS plus a vector/semantic layer (`pgvector`)
- Pros: semantic product discovery.
- Cons: a cashier searching "coke" wants exact matching, fast, not semantic neighbours.
- Rejected for now; `pgvector` is available in-database if a discovery feature later
  justifies it, with no new service.

## Consequences

- No new infrastructure, no sync pipeline, no stale-index class of bug.
- Search results are transactionally consistent with the data — a product created in a
  transaction is searchable the moment that transaction commits.
- `tsvector` maintenance is a write-time cost on catalog updates. Negligible at
  master-data write rates.
- Language configuration is per-tenant (`english`, `simple`, etc.) and must be chosen at
  setup; changing it later requires reindexing.
- Relevance tuning is limited to weights and ranking functions. There is no learning-to-rank
  and no synonym management beyond a manually maintained dictionary.
- The client and server search implementations must be held in sync by shared test data.

## Revisit when

- A single tenant's catalog exceeds roughly one million rows.
- Cross-tenant or cross-store federated search becomes a product feature.
- Relevance tuning becomes a product feature rather than a setting.
- Faceted navigation with counts is required.
- Search latency at p99 exceeds 200 ms on realistic data despite indexing.

**Exit path:** search already sits behind a `SearchService` interface per module. Swapping
the implementation means adding an indexing consumer that reads the existing event log
with its own cursor — the log is already there, so no new write-path plumbing is needed.

## Related
- [Architecture §2.6](../architecture/02-server.md#26-search), [ADR-0004](0004-replication-tiers.md)
