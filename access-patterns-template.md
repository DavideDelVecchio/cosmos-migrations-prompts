# Access Patterns

> Fill in the tables below with the known access patterns for your application.
> This information helps the migration assistant design optimal Cosmos DB containers,
> partition key strategies, and indexing policies.
>
> **TPS** = Transactions Per Second — estimated number of times this pattern executes per second under normal load.
>
> **Latency Requirement** — the maximum acceptable end-to-end latency for this operation from the application's perspective.
>
> **Tip:** This file is the primary source for access pattern analysis during Cosmos DB container design.
> Attach additional files (Cassandra trace logs, query analytics, APM dashboards) if available.

---

## Read Patterns

| #    | Pattern Name                  | Tables / Entities            | Filter / Lookup Fields                  | Sort Order          | Paginated? | Frequency (TPS) | Latency Requirement | Cassandra Feature Used     | Notes                                                     |
|-----:|-------------------------------|------------------------------|-----------------------------------------|---------------------|-----------|----------------:|---------------------|---------------------------|-----------------------------------------------------------|
| R001 | `<pattern name>`              | `<table-a>`                  | `<partition-key-field>`                 | N/A                 | No        |                 | `< N ms`            | Plain CQL — point read    | `<e.g., most latency-sensitive operation>`                |
| R002 | `<pattern name>`              | `<table-a>`, `<table-b>`     | `<foreign-key-field>`, `<date-field>`   | `<field>` DESC      | Yes       |                 | `< N ms`            | Plain CQL — range scan    | `<e.g., paginated, newest-first; continuation token>`     |
| R003 | `<pattern name>`              | `<table-c>`                  | `<search-fields>`                       | Relevance score     | Yes       |                 | `< N ms`            | DSE Search / Solr         | `<e.g., full-text + prefix autocomplete>`                 |
| R004 | `<pattern name>`              | `<table-a>` (multi-get)      | `<list of partition keys>`              | N/A                 | No        |                 | `< N ms`            | Plain CQL — IN clause     | `<e.g., bulk point-read for N videoIds>`                  |
| R005 | `<pattern name>`              | `<graph traversal>`          | `<entity-id>` — graph edge walk         | N/A                 | No        |                 | `< N ms`            | DSE Graph / Gremlin       | `<e.g., collaborative-filter recommendation traversal>`   |

> **Add or remove rows** to match all read patterns in your application.

---

## Write Patterns

| #    | Pattern Name                  | Tables / Entities            | Single / Batch      | Conditional?       | Frequency (TPS) | Latency Requirement | Cassandra Feature Used     | Notes                                                     |
|-----:|-------------------------------|------------------------------|---------------------|--------------------|----------------:|---------------------|---------------------------|-----------------------------------------------------------|
| W001 | `<pattern name>`              | `<table-a>`, `<table-b>`     | Batch (N inserts)   | No                 |                 | `< N ms`            | LOGGED BATCH              | `<e.g., entity created in 2 tables simultaneously>`       |
| W002 | `<pattern name>`              | `<table-a>`                  | Single              | IF NOT EXISTS (LWT)|                 | `< N ms`            | Lightweight Transaction   | `<e.g., email uniqueness enforced at write time>`         |
| W003 | `<pattern name>`              | `<table-b>`                  | Single (counter)    | No                 |                 | `< N ms`            | Counter column increment  | `<e.g., atomic view count increment, high write TPS>`     |
| W004 | `<pattern name>`              | `<table-c>`                  | Batch (cleanup)     | No                 |                 | `< N ms`            | TTL (auto-expiry)         | `<e.g., time-windowed feed; TTL handles deletion>`        |
| W005 | `<pattern name>`              | `<table-a>`                  | Single (update)     | No                 |                 | `< N ms`            | Plain CQL — UPDATE        | `<e.g., partial field update on existing record>`         |

> **Add or remove rows** to match all write patterns in your application.

---

## Cosmos DB Query Mapping

> For each read pattern above, document the equivalent Cosmos DB SQL query or SDK call.
> This section is populated by the migration assistant during Prompt 2 (Schema & Access Patterns Conversion Plan).

| Pattern # | Cosmos DB Query / Operation | Partition Key Hit? | Index Type Used | Notes |
|----------:|-----------------------------|--------------------|-----------------|-------|
| R001 | `container.readItem(id, partitionKey)` | ✅ Yes — point read | N/A | ~1 RU |
| R002 | `SELECT * FROM c WHERE c.<fk> = @fk ORDER BY c.<ts> DESC` | ✅ Yes | Composite index on `(<fk> ASC, <ts> DESC)` | Requires composite index; paginate with continuation token |
| R003 | `FullTextContains(c.<field>, @q)` OR Azure AI Search | ✅ Yes (AI Search) | Full-text index on `/<field>` | Cosmos DB FTS (BM25) or Azure AI Search external index |
| R004 | N point reads in parallel via SDK | ✅ Yes (each) | N/A | Use `Flux.merge()` or `Task.WhenAll()` for parallel reads |
| R005 | Pre-computed: `SELECT * FROM c WHERE ARRAY_CONTAINS(c.recommendedIds, @id)` | ✅ Yes | Array index on `/recommendedIds` | Replace graph traversal with pre-computed list in entity doc |
| W001 | `container.executeCosmosBatch(batch)` | ✅ Yes — same PK | N/A | `TransactionalBatch` requires all docs share partition key |
| W002 | `container.createItem(item, options)` with `IfNoneMatchETag("*")` | ✅ Yes | N/A | Throws `CosmosException(409)` on duplicate — equivalent to LWT |
| W003 | `container.patchItem(id, pk, CosmosPatchOperations.create().increment("/field", 1))` | ✅ Yes | N/A | Atomic — no read-modify-write cycle needed |
| W004 | Set `"ttl": <N>` on document; container `defaultTtl: -1` | ✅ Yes | N/A | Cosmos DB TTL daemon handles deletion automatically |
| W005 | `container.patchItem(id, pk, ops)` or `container.upsertItem(doc)` | ✅ Yes | N/A | Patch for partial update; upsert for full replacement |

---

## Partition Key Design Review

> For each planned Cosmos DB container, validate the partition key choice against the access patterns above.

| Container | Chosen Partition Key | Hot Partition Risk | Cross-Partition Queries? | Justification |
|-----------|---------------------|--------------------|--------------------------|---------------|
| `<container-1>` | `/<field>` | Low — high cardinality UUID | No — all queries filter on PK | `<reason>` |
| `<container-2>` | `/<field>` | Medium — date-bucketed | No — date bucket is the PK | `<reason>` |
| `<container-3>` | `/<field>` | Low | Yes — per-user activity feed requires fan-out | `<trade-off accepted because low TPS>` |

---

## Additional Notes

> Add any extra context about access patterns, batch operations, reporting queries, caching layers, or SLA requirements.

- **Caching:** `<e.g., Redis cache in front of Cosmos DB for high-TPS read patterns R001, R004>`
- **Reporting / analytics:** `<e.g., Synapse Link enabled for analytical queries — does not consume RU/s>`
- **Change feed consumers:** `<e.g., Azure Function on change feed updates search index and recommendation arrays>`
- **Consistency level:** `<e.g., Session consistency for user-facing flows; Eventual for feed/stats aggregations>`
