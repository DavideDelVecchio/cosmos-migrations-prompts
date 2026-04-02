# Volumetrics

> Fill in the table below with volumetric data for each Cassandra table or logical entity in your database.
> This information helps the migration assistant estimate request unit (RU) consumption,
> design partition strategies, and choose the appropriate Cosmos DB capacity mode
> (Serverless vs. Autoscale vs. Provisioned).
>
> Leave cells empty if you don't have estimates — the migration assistant will infer missing values
> from schema analysis and sample data (if provided).
>
> **TPS** = Transactions Per Second — estimated read/write operations per second under normal (steady-state) load.
> **RU** = Request Units — Cosmos DB's normalized throughput unit (a 1 KB point read costs ~1 RU).
>
> **Tip:** This file is the primary source for volumetric data during Cosmos DB container design.
> Attach additional files (CSV exports, AWR/ASH reports, Cassandra `nodetool tpstats` output,
> Prometheus metrics) alongside this template if available.

---

## Table Volumetrics

| #   | Keyspace / Schema | Table / Entity       | Est. Row Count | Avg Row Size (KB) | Growth Rate      | Read TPS | Write TPS | DSE Feature        | Notes                                           |
|----:|-------------------|----------------------|---------------:|------------------:|------------------|---------:|----------:|--------------------|--------------------------------------------------|
| V1  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | Plain CQL          | `<e.g., partition key = userId, point-read heavy>` |
| V2  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | Counter columns    | `<e.g., monotonic increment, high write TPS>`   |
| V3  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | DSE Search / Solr  | `<e.g., full-text queries on name, description>`|
| V4  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | Plain CQL + TTL    | `<e.g., 7-day rolling window, auto-expiry>`     |
| V5  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | Plain CQL + LWT    | `<e.g., uniqueness check on email>`             |
| V6  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | Batch (dual-write) | `<e.g., same entity written to 2 tables>`       |
| V7  | `<keyspace>`      | `<table>`            |                |                   | `<N>% / month`   |          |           | DSE Graph          | `<e.g., graph vertices/edges for recommendations>`|

> **Add or remove rows** to match the actual number of tables in your keyspace.

---

## Cosmos DB RU Budget Guidance

Use the estimated TPS and row sizes above to project peak RU consumption per container:

| Cosmos DB Container | Partition Key | Mapped From (Cassandra Tables) | Est. Read RU/s | Est. Write RU/s | Capacity Mode Recommendation |
|--------------------|--------------|-------------------------------|---------------:|----------------:|------------------------------|
| `<container-1>`    | `/<field>`   | `<table-a>`, `<table-b>`      |                |                 | `<Serverless / Autoscale / Provisioned>` |
| `<container-2>`    | `/<field>`   | `<table-c>`                   |                |                 | `<Serverless / Autoscale / Provisioned>` |

> **Cosmos DB RU estimation rules of thumb:**
> - Point read (1 KB document): ~1 RU
> - Point write / upsert (1 KB document): ~5 RU
> - Cross-partition query (fan-out): 1 RU per physical partition scanned × query complexity
> - Patch API (single field increment): ~5–10 RU depending on document size
> - Transactional Batch (N operations): sum of individual operation RU costs

---

## Retention & TTL Planning

| Table / Container | Retention Requirement | Cassandra TTL (seconds) | Cosmos DB TTL Setting |
|------------------|----------------------|------------------------|----------------------|
| `<table>`        | `<N> days / years`   | `<N>` (or none)        | `"ttl": <N>` on document; container `defaultTtl: -1` |
| `<table>`        | Indefinite           | None                   | No TTL on container  |

---

## Traffic Patterns & Additional Notes

> Add any context about traffic patterns, seasonal spikes, batch job load, or operational constraints.

- **Peak hours:** `<e.g., 09:00–11:00 UTC weekdays; 3× normal TPS>`
- **Batch jobs:** `<e.g., nightly export job runs at 02:00 UTC, generating N write TPS for 30 min>`
- **Seasonal spikes:** `<e.g., 10× spike during Black Friday / product launch>`
- **Geographic distribution:** `<e.g., 80% US East, 20% Europe>`
- **Data retention policy:** `<e.g., transactional data kept 2 years; analytics data kept 7 years>`
- **Consistency SLA:** `<e.g., read-your-writes required for user-facing flows; eventual acceptable for feed/stats>`
