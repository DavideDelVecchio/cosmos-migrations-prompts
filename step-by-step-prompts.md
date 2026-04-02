# Cassandra → Azure Cosmos DB NoSQL Migration: Step-by-Step Prompts

This document provides a generic, reusable sequence of prompts for migrating any Apache Cassandra (or DataStax Enterprise) application to Azure Cosmos DB NoSQL using GitHub Copilot as an AI coding agent.

> **How to use:** Copy each prompt verbatim into your agent chat, substituting `<placeholders>` with your project's values. Each prompt builds on the outputs of the previous one.

---

## Prompt 1 — Migration Assessment Report

> **Prompt:** Generate an assessment report for migrating `<your-application-name>` from its current Cassandra (or DataStax Enterprise) database backend to Azure Cosmos DB NoSQL.
>
> Include analysis of:
> - The current data model (keyspaces, tables, column types, partition keys, clustering columns)
> - Access patterns (read/write queries by service or module)
> - DSE-specific features in use (DSE Search/Solr, DSE Graph/Gremlin, counter columns, LWT, TTL, batch statements, paging)
> - Runtime and test dependencies
> - Migration risks (architecture, data model, consistency, operations)
>
> Use `access-patterns-template.md` and `volumetrics-template.md` to document discovered access patterns and volumetric projections based on sample data (use `<N>` years retention horizon).

**Expected outputs:**
```
docs/cosmos-db-migration-assessment.md
docs/access-patterns.md
docs/volumetrics.md
```

---

## Prompt 2 — Schema & Access Patterns Conversion Plan

> **Prompt:** Based on the assessment report, create a detailed schema and access patterns conversion plan for Cosmos DB NoSQL. Design the document models, partition keys, container strategy, indexing policies, and query mappings.
>
> - Identify any tables that are **out of scope** (e.g., operational/metadata tables) and explicitly exclude them.
> - For each entity, evaluate whether to embed, reference, or denormalize related data.
> - Evaluate combining related tables into a single container (multi-entity container pattern) where access patterns allow.
> - Map each Cassandra CQL query to an equivalent Cosmos DB SQL query or point-read pattern.
> - Document partition key choices with justification (cardinality, query alignment, hot-partition risk).

**Expected output:**
```
docs/schema_and_access_patterns_conversion_plan.md
```

---

## Prompt 3 — Bicep Infrastructure-as-Code

> **Prompt:** Based on `schema_and_access_patterns_conversion_plan.md`, generate Bicep infrastructure-as-code to provision the Cosmos DB account, database, and containers with the designed schema, partition keys, and indexing policies.
>
> - Use Entra ID authentication (`DefaultAzureCredential`) — no connection strings or primary keys.
> - Grant standard Control Plane Operator and Data Plane Contributor RBAC roles to the specified principal.
> - Set container-level indexing policies to match the access patterns (include/exclude paths as needed).
> - Use item-level TTL (with `defaultTtl: -1` on the container) for any tables that require row expiry.

**Expected outputs:**
```
infra/main.bicep        — Cosmos DB account, database, containers, indexing policies, RBAC role assignments
infra/main.bicepparam   — Deployment parameters
```

**Include the following variables in the Bicep parameter file** (`infra/main.bicepparam`):
```bicep
param location = '<azure-region>'                  // e.g., 'eastus2'
param cosmosAccountName = '<cosmos-account-name>'  // e.g., 'myapp-cosmos-01'
param databaseName = '<database-name>'             // e.g., 'myapp'
param principalId = '<entra-principal-id>'         // e.g., '00000000-0000-0000-0000-000000000000'
param tags = {
  owner: '<owner-alias>'
}
```

**Use the following parameters for deployment script generation:**
```
resource-group: '<your-resource-group>'
```

---

## Prompt 4 — Deploy Infrastructure to Azure

> **Prompt:** Deploy the Bicep infrastructure to Azure to provision the Cosmos DB account, database, and containers.
>
> - Create the resource group if it does not exist.
> - Deploy `infra/main.bicep` at the resource group scope with all required parameters from `infra/main.bicepparam`.
> - Verify that the Cosmos DB account, database, and all containers are provisioned with the correct partition keys, indexing policies, and RBAC role assignments.
> - Output the deployment and verification commands and results.

**Deployment command:**
```bash
az deployment group create \
  --resource-group <resource-group-name> \
  --template-file infra/main.bicep \
  --parameters infra/main.bicepparam
```

---

## Prompt 5 — Data Migration

> **Prompt:** Migrate data from the source Cassandra/DSE cluster to the target Cosmos DB containers.
>
> - Export data from each Cassandra table using `<your-export-tool>` (e.g., `cqlsh COPY`, DataStax Spark Connector, or custom script).
> - Transform each row to the Cosmos DB document model defined in `schema_and_access_patterns_conversion_plan.md` — add `id`, `type`, and any partition key fields required by the container design.
> - Load transformed documents into the target containers using the Cosmos DB Bulk Executor SDK or Azure Data Factory.
> - Validate document counts match expected volumetrics from `volumetrics.md`.
>
> Save all generated transformation scripts or tools in `/DataMigration/tools/` and intermediate JSON files in `/DataMigration/data/` before loading.

---

## Prompt 6 — Application Conversion Plan

> **Prompt:** Read all source code files (DAO/repository classes, service classes, models/entities, configuration, application entry point) and `schema_and_access_patterns_conversion_plan.md`, then generate a comprehensive application conversion plan detailing every code change needed to migrate the application from the Cassandra driver to the Azure Cosmos DB SDK.
>
> The plan must explicitly address:
>
> - **Driver replacement:** Remove the Cassandra/DSE driver dependency; replace with the Azure Cosmos DB SDK for `<your-language>` (Java SDK v4, .NET SDK v3, Python SDK, etc.).
> - **Authentication:** Switch all database connections from username/password or `application.conf` to `DefaultAzureCredential` (Entra ID, no secrets in code or config).
> - **Session/Client factory:** Replace the Cassandra `Session` singleton with a `CosmosAsyncClient` (or sync equivalent) singleton configured via the SDK builder.
> - **DAO/Repository rewrite:** Replace each `Mapper<T>` / `CqlSession.execute()` call with Cosmos DB SDK point reads, queries (`SqlQuerySpec`), and upserts.
> - **Paging:** Replace Cassandra `PagingState` (opaque bytes) with Cosmos DB string continuation tokens.
> - **Batch writes:** Replace Cassandra `BatchStatement` with Cosmos DB `TransactionalBatch` (same-partition) or multi-document upsert loops.
> - **Counter columns:** Replace `QueryBuilder.incr(...)` with Cosmos DB Patch API `PatchOperationType.INCREMENT`.
> - **Lightweight Transactions (LWT):** Replace `INSERT ... IF NOT EXISTS` with a conditional create using `IfNoneMatchETag("*")`.
> - **Full-text search (if applicable):** Replace DSE Search/Solr queries with Azure AI Search (preferred) or Cosmos DB built-in full-text search.
> - **Graph traversals (if applicable):** Replace DSE Graph/Gremlin traversals with a pre-computed recommendation array, Cosmos DB for Apache Gremlin, or Azure AI Personalizer.
> - **TTL:** Replace Cassandra row-level TTL with Cosmos DB item-level `"ttl"` property (integer seconds).
> - **Async model:** Align async primitives — e.g., Guava `ListenableFuture` → Project Reactor `Mono/Flux` (Java); `Task<T>` → Cosmos SDK `Task<T>` (.NET); callbacks → `asyncio` (Python).
> - **Test infrastructure:** Replace `cassandra-unit` / embedded Cassandra with Azure Cosmos DB Emulator (Docker) for integration tests.

**Expected output:**
```
docs/application_conversion_plan.md
```

---

## Prompt 7 — Execute Application Conversion

> **Prompt:** Following the final `schema_and_access_patterns_conversion_plan.md` and `application_conversion_plan.md`, rewrite the application DAOs/repositories for Cosmos DB NoSQL, start the application, and run API validation tests.
>
> **Success Criteria:**
> - Goal 1 — Successful build with 0 errors
> - Goal 2 — Application starts successfully and is reachable (gRPC port / HTTP port / local URL)
> - Goal 3 — All API validation tests pass
>
> **API Validation Test Requirements:**
> - **Dynamic entity discovery:** Tests must discover entity IDs from list/index endpoints rather than hardcoding assumptions about migrated data ordering.
> - **Protocol awareness:** Use the correct client (gRPC client, HTTP client, etc.) matching the application's wire protocol.
> - **Environment config:** Read endpoint/port from application config (`application.properties`, `appsettings.json`, `launchSettings.json`, etc.) rather than assuming fixed values.

**Goal 3 — Validation Test Results:**

| # | Test | Endpoint / Method | Status | Verified |
|---|------|-------------------|--------|----------|
| 1 | `<test name>` | `<endpoint>` | ⬜ PASS / FAIL | `<what was verified>` |
| 2 | `<test name>` | `<endpoint>` | ⬜ PASS / FAIL | `<what was verified>` |
| … | … | … | … | … |

---

## Technology Stack Reference

Fill in this table at the end of the migration to record the final state:

| Component | Value |
|-----------|-------|
| Application framework | `<e.g., Spring Boot 3.x / ASP.NET Core 9 / FastAPI>` |
| Source database driver | `<e.g., DSE Java Driver 1.8.2 / DataStax C# Driver 3.x>` |
| Target Cosmos DB SDK | `<e.g., azure-cosmos 4.x (Java) / Microsoft.Azure.Cosmos 3.x (.NET)>` |
| Authentication | `DefaultAzureCredential` (Entra ID, no keys) |
| Cosmos DB account | `<cosmos-account-name>.documents.azure.com:443/` |
| Database name | `<database-name>` |
| Containers | `<container-1>` (PK `/<field>`), `<container-2>` (PK `/<field>`), … |
| Capacity mode | `<Serverless / Autoscale / Provisioned>` |
| Infrastructure | Bicep (`infra/main.bicep` + `infra/main.bicepparam`) |
| Data migration tool | `<e.g., Spark Connector / ADF / custom Node.js script>` |
| RBAC principal | `<entra-principal-id>` |
| Entra ID tenant | `<tenant-id>` |
