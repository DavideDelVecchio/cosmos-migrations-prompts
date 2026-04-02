# `<Application Name>` → Azure Cosmos DB NoSQL: Migration Assessment Report

> Generated: `<YYYY-MM-DD>` | Source: `<Application Name>` — `<Source DB Technology>` (e.g., Apache Cassandra 4.x / DataStax Enterprise 6.x)

---

## 1. Executive Summary

`<Application Name>` is a `<brief application description>`. The backend `<exposes N services / serves N controllers / etc.>`, each owning a slice of the `<keyspace-name>` Cassandra keyspace.

> **Tip:** Describe in 2–3 sentences: what the application does, how many Cassandra tables it uses, and which DSE-specific capabilities it exploits (Search/Solr, Graph/Gremlin, counter columns, LWT, TTL, batch).

Migration to Azure Cosmos DB NoSQL is **`<Feasible / Feasible with high effort / Not recommended>`** with `<low / moderate / significant>` effort. The core CRUD workloads map cleanly to Cosmos DB point reads and single-partition queries. `<Describe any DSE-specific subsystems that require architectural substitution.>`

**Overall Migration Complexity: `<LOW / MEDIUM / HIGH / CRITICAL>`**

- `<N>` Cassandra tables across `<N>` services/modules
- `<Describe which patterns map cleanly>`
- `<Describe which DSE-specific features require replacement>`
- `<Describe any counter / LWT / batch concerns>`
- `<Describe messaging or other layers that are unaffected>`

**Estimated effort:** `<N–M weeks>` for a team of `<N>` engineers.

---

## 2. Current Architecture

### 2.1 Technology Stack

| Component | Current Version | Notes |
|-----------|-----------------|-------|
| JVM / Runtime | `<Java 11 / .NET 8 / Python 3.12 / etc.>` | |
| Framework | `<Spring Boot 2.7.x / ASP.NET Core 8 / FastAPI / etc.>` | |
| Database driver | `<DSE Java Driver 1.8.x / DataStax C# Driver 3.x / cassandra-driver-core 4.x>` | |
| Wire protocol | `<gRPC 1.x / REST / GraphQL>` | |
| Messaging | `<Kafka / RabbitMQ / Azure Service Bus / in-memory>` | |
| Search | `<DSE Search (Solr) / Elasticsearch / None>` | |
| Graph | `<DSE Graph (TinkerPop/Gremlin) / Neo4j / None>` | |
| Build tool | `<Maven / Gradle / dotnet / poetry>` | |

### 2.2 Module / Service Map

| Module / Service | Cassandra Tables Owned | DSE Feature Used |
|-----------------|----------------------|-----------------|
| `<module-1>` | `<table-a>`, `<table-b>` | `<Plain CQL / DSE Search / Counter / LWT / etc.>` |
| `<module-2>` | `<table-c>` | `<Plain CQL / DSE Graph / etc.>` |

---

## 3. Current Data Model

### 3.1 Cassandra Keyspace: `<keyspace-name>`

> **Design note:** Cassandra schemas are explicitly denormalized by query shape. Document each table separately, including which tables represent the same logical entity stored in multiple forms to serve different query patterns.

<!-- Repeat the table block below for each Cassandra table -->

#### Table: `<table-name>` — `<one-line purpose>`

| Column | Type | Role |
|--------|------|------|
| `<col>` | `<type>` | **Partition key** |
| `<col>` | `<type>` | **Clustering col N** (`<ASC / DESC>`) |
| `<col>` | `<type>` | `<data column description>` |

<!-- Add additional tables as needed -->

### 3.2 DSE-Specific Features (beyond plain Cassandra)

> Remove rows that do not apply to your application.

| Feature | Where Used | Migration Impact |
|---------|-----------|-----------------|
| **DSE Search (Solr)** | `<DAO class / service>` — `<query description>` | **HIGH** — Replace with Azure AI Search or Cosmos DB built-in full-text search |
| **DSE Graph (Gremlin)** | `<DAO class / service>` — `<traversal description>` | **CRITICAL** — Replace with Cosmos DB for Apache Gremlin or pre-computed recommendations |
| **Counter columns** | `<table-name>` — `<column names>` | **HIGH** — Replace with Cosmos DB Patch API atomic increment |
| **Lightweight Transactions (LWT)** | `<DAO / table>` — `INSERT ... IF NOT EXISTS` | **MEDIUM** — Replace with Cosmos DB conditional create (`IfNoneMatchETag: *`) |
| **Batch statements** | `<DAO / tables>` — `<description>` | **MEDIUM** — Use Cosmos DB Transactional Batch (same-partition) or eventual consistency |
| **TTL** | `<table-name>` — `<N>` seconds item expiry | **LOW** — Use Cosmos DB item-level TTL (`"ttl": <N>` per document) |
| **PagingState** | `<services/methods>` — opaque byte paging token | **LOW** — Replace with Cosmos DB string continuation token |

---

## 4. Access Patterns Analysis

See [access-patterns.md](../access-patterns.md) for the full enumeration. The table below summarizes service-level coverage:

| Service / Module | Primary Read Pattern | Primary Write Pattern | Special Requirement |
|-----------------|--------------------|-----------------------|---------------------|
| `<service-1>` | `<e.g., Point read by entity ID>` | `<e.g., Insert with LWT>` | `<e.g., Uniqueness constraint>` |
| `<service-2>` | `<e.g., List by foreign key, paginated>` | `<e.g., Batch insert across N tables>` | `<e.g., Fan-out write>` |
| `<service-3>` | `<e.g., Full-text search by keyword>` | None (read-only) | `<e.g., DSE Solr — Azure AI Search required>` |

---

## 5. Dependencies Analysis

### 5.1 Runtime Dependencies

| Dependency | Version | Migration Impact |
|-----------|---------|-----------------|
| `<cassandra/dse-driver>` | `<version>` | **Replace entirely** — migrate to Azure Cosmos DB SDK |
| `<framework>` | `<version>` | No change — unaffected by database swap |
| `<messaging-lib>` | `<version>` | No change — async messaging layer unaffected |
| `<other-dep>` | `<version>` | No change |

### 5.2 Driver Components to Replace

| Cassandra/DSE Component | Class / API | Cosmos DB Equivalent |
|------------------------|-------------|---------------------|
| Session factory | `<DseSession / CqlSession>` | `CosmosAsyncClient` singleton via SDK builder |
| ORM / Mapper | `<MappingManager / Mapper<T>>` | Jackson `ObjectMapper` + SDK serialization (Java); `System.Text.Json` (.NET) |
| DAO base class | `<DseDaoSupport / RepositoryBase>` | Custom container-support base class |
| Paging | `PagingState` (opaque bytes) | `CosmosQueryRequestOptions` continuation token (`String`) |
| Batch writes | `BatchStatement` | `CosmosBatch` (same-partition) |
| Prepared statements | `PreparedStatement` / `BoundStatement` | `SqlQuerySpec` with named parameters |
| Graph API | `GraphStatement` / Gremlin traversal | Azure Cosmos DB for Apache Gremlin (separate account) |
| Solr full-text query | `solr_query` field via CQL | Azure AI Search index + Cosmos DB change feed indexer |
| Counter increment | `QueryBuilder.incr(...)` | `PatchOperations.increment("/field", 1)` |
| LWT (`IF NOT EXISTS`) | `Statement.setSerialConsistencyLevel(SERIAL)` | `IfNoneMatchEtag: *` condition on create |

### 5.3 Test Dependencies

| Test Dependency | Version | Migration Strategy |
|----------------|---------|-------------------|
| `cassandra-unit` / `testcontainers-cassandra` | `<version>` | Replace with Azure Cosmos DB Emulator (Docker image) |
| `cassandra-all` (embedded Cassandra) | `<version>` | Remove — no longer needed |
| `<test-framework>` | `<version>` | Keep |
| `<assertion-library>` | `<version>` | Keep |

---

## 6. Migration Risks

### 6.1 Risk Matrix

| # | Risk | Category | Severity | Likelihood | Mitigation |
|---|------|----------|----------|-----------|------------|
| R1 | DSE Graph replacement — no Gremlin traversal in Cosmos DB NoSQL API | Architecture | **CRITICAL** | Certain | Option A: Cosmos DB for Apache Gremlin (separate account). Option B: Pre-compute recommendations into entity documents via Azure Function. Option C: Azure AI Personalizer / AI Foundry for ML-based recommendations. |
| R2 | DSE Search (Solr) replacement — Cosmos DB NoSQL `CONTAINS` is not full-text | Architecture | **HIGH** | Certain | Primary: Azure AI Search with Cosmos DB change-feed indexer. Secondary: Cosmos DB built-in full-text search (BM25, generally available). |
| R3 | Counter columns — no native counter type in Cosmos DB | Data Model | **HIGH** | Certain | Use Cosmos DB Patch API `PatchOperationType.INCREMENT` — fully atomic, no read-modify-write required. |
| R4 | Dual-write atomicity (fan-out writes) — loses Cassandra `LOGGED BATCH` semantics | Data Model | **HIGH** | Medium | Use Cosmos DB Transactional Batch when all documents share the same partition key. Otherwise: accept eventual consistency + async compensation via change feed. |
| R5 | LWT for uniqueness (`IF NOT EXISTS`) | Consistency | **MEDIUM** | Certain | Cosmos DB conditional create with `IfNoneMatchETag = "*"` — throws `409 Conflict` on duplicate, functionally equivalent. |
| R6 | Row-level TTL | Data Model | **MEDIUM** | Certain | Set per-document `"ttl": <N>` property. Enable default TTL on container with `-1` (inherit from document). |
| R7 | Time-based clustering key ordering (TimeUUID) | Data Model | **MEDIUM** | Certain | Replace TimeUUID with ISO-8601 timestamp string; use `ORDER BY c.<timestampField> DESC` in Cosmos DB SQL. |
| R8 | Cassandra `PagingState` exposed in APIs | API | **LOW** | Certain | Surface Cosmos DB string continuation token in API response instead of opaque bytes. |
| R9 | Async model mismatch | Code | **MEDIUM** | Certain | Java: Guava `ListenableFuture` → Project Reactor `Mono/Flux`. .NET: already `Task<T>`. Python: callbacks → `asyncio`. |
| R10 | Live data migration — DSE data must be extracted before cutover | Operations | **HIGH** | High | Apache Spark + DataStax Spark Connector to bulk-read DSE → transform → bulk-write via Azure Data Factory or Cosmos Bulk Executor. |
| R11 | Embedded Cassandra in tests (`cassandra-unit`) | Testing | **LOW** | Certain | Replace with Azure Cosmos DB Emulator (Docker) for integration tests. |

> **Add or remove rows** to match risks actually present in your application.

### 6.2 Deep Dive: Critical Risks

#### R1 — Graph Recommendations (if applicable)

> Describe the specific graph traversal query used. Provide replacement options with fidelity/complexity/cost trade-off table (see example below).

| Option | Fidelity | Complexity | Cost |
|--------|----------|-----------|------|
| Cosmos DB for Apache Gremlin | High (same traversal logic) | High (separate account, change feed sync) | Medium |
| Pre-computed recommendation array in entity document | Medium (batch vs. real-time) | Low (Azure Function on events) | Low |
| Azure AI Personalizer / AI Foundry | Highest (ML-based) | Medium-High (API integration) | High |

#### R2 — Full-Text Search (if applicable)

> Describe the specific Solr/Lucene query. Map each search field to an Azure AI Search field configuration (searchable, filterable, facetable, sortable).

#### R3 — Counter Columns (if applicable)

> Show the original Cassandra counter increment CQL and the equivalent Cosmos DB Patch API call.

---

## 7. Proposed Cosmos DB Container Strategy

| Container | Partition Key | Entities Stored | Design Decision |
|-----------|--------------|----------------|----------------|
| `<container-1>` | `/<field>` | `<entity-a>` | `<justification>` |
| `<container-2>` | `/<field>` | `<entity-b>` (`type:"<type-1>"`), `<entity-c>` (`type:"<type-2>"`, TTL `<N>` s) | `<justification>` |
| `<container-3>` | `/<field>` | `<entity-d>` aggregate + `<entity-e>` per-user entries | `<justification — e.g., Transactional Batch eligible>` |

---

## 8. Migration Phases

### Phase 1 — Preparation & Architecture Decisions (Week 1–`<N>`)
- Abstract all DAO/repository interfaces from driver-specific implementations
- Decide on Graph replacement strategy (pre-computed vs. Gremlin API)
- Decide on Search replacement (Azure AI Search vs. built-in Cosmos full-text search)
- Provision Cosmos DB account + containers (Bicep IaC)
- Set up Cosmos DB Emulator for local developer testing

### Phase 2 — Service Migration (Week `<N>`–`<M>`, ordered by risk level)
1. **`<simplest service>`** — `<reason it is lowest risk>`
2. **`<next service>`** — `<description>`
3. …
4. **`<search service>`** — Azure AI Search integration
5. **`<graph/recommendation service>`** — Graph/recommendation replacement

### Phase 3 — Data Migration (Week `<N>`–`<M>`, parallel with Phase 2)
- Export from Cassandra/DSE using `<tool>`
- Transform rows to Cosmos DB document shape (add `id`, `type`, partition key fields)
- Bulk-load via Azure Data Factory or Cosmos DB Bulk Executor SDK
- Validate document counts and data integrity

### Phase 4 — Dual-Write & Cutover (Week `<N>`–`<M>`)
- Run dual-write mode (write to both Cassandra and Cosmos DB simultaneously)
- Validate reads from Cosmos DB match Cassandra
- Switch all service DAOs to Cosmos DB read path
- Decommission Cassandra/DSE cluster

---

## 9. Effort Estimate

| Activity | Effort (person-days) |
|----------|---------------------|
| Container design + Bicep IaC | `<N>` |
| SDK integration & base DAO/container support class | `<N>` |
| `<service-1>` migration | `<N>` |
| `<service-2>` migration | `<N>` |
| `<service-N>` migration | `<N>` |
| Search replacement (Azure AI Search) | `<N>` |
| Graph/recommendation replacement | `<N>` |
| Data migration pipeline | `<N>` |
| Integration testing + Cosmos DB Emulator setup | `<N>` |
| Cutover + validation | `<N>` |
| **Total** | **`<N>` person-days (~`<N–M>` weeks, `<N>` engineers)** |

---

## 10. Expected Benefits After Migration

| Benefit | Detail |
|---------|--------|
| **Managed service** | No Cassandra/DSE cluster operations, automatic patching, built-in backups |
| **Elastic scale** | Autoscale RU/s — no provisioning for peak; Serverless option for dev/staging |
| **Global distribution** | Multi-region active-active writes with < 10 ms p99 reads globally |
| **Integrated security** | Entra ID RBAC, customer-managed keys, VNet integration, private endpoints |
| **Full-text search parity** | Azure AI Search provides Solr-equivalent full-text, facets, and autocomplete |
| **Reduced CVE surface** | Eliminates proprietary/EOL Cassandra driver with no public CVE remediation path |
| **Cost optimization** | Serverless or autoscale RU/s vs. always-on Cassandra nodes |

---

## 11. References

- [Azure Cosmos DB for NoSQL documentation](https://learn.microsoft.com/azure/cosmos-db/nosql/)
- [Azure Cosmos DB Java SDK v4](https://learn.microsoft.com/azure/cosmos-db/nosql/sdk-java-v4)
- [Azure Cosmos DB .NET SDK v3](https://learn.microsoft.com/azure/cosmos-db/nosql/sdk-dotnet-v3)
- [Cosmos DB Patch API — partial document update](https://learn.microsoft.com/azure/cosmos-db/partial-document-update)
- [Cosmos DB Transactional Batch](https://learn.microsoft.com/azure/cosmos-db/nosql/transactional-batch)
- [Azure Cosmos DB for Apache Gremlin](https://learn.microsoft.com/azure/cosmos-db/gremlin/)
- [Azure AI Search + Cosmos DB indexer](https://learn.microsoft.com/azure/search/search-howto-index-cosmosdb)
- [Cosmos DB Change Feed](https://learn.microsoft.com/azure/cosmos-db/change-feed)
- [Azure Cosmos DB Emulator (Docker)](https://learn.microsoft.com/azure/cosmos-db/emulator)
- [Cosmos DB Full-Text Search](https://learn.microsoft.com/azure/cosmos-db/gen-ai/full-text-search)
