# trinodb/trino

> A distributed SQL query engine that runs interactive analytics across data where it already lives — no storage of its own.

[GitHub repo](https://github.com/trinodb/trino) ·
[Official website](https://trino.io) ·
[License: Apache-2.0](https://github.com/trinodb/trino/blob/master/LICENSE)

## Overview

Trino is a massively parallel (MPP) SQL engine for querying large datasets that sit in other systems — object storage data lakes, Hive/Iceberg/Delta tables, relational databases, and dozens of other sources — without moving the data first. It has no storage layer: a Trino cluster is pure compute that federates queries over pluggable connectors. Its home is interactive, ad-hoc analytics over big data, where sub-minute latency on terabyte scans matters more than transactional guarantees.

The project is the direct descendant of Presto, created at Facebook in 2012 and open-sourced in 2013. In 2018 the three original authors (Martín Traverso, Dain Sundstrom, David Phillips) left Facebook and forked the codebase as PrestoSQL under the newly formed Presto Software Foundation. A trademark dispute over the "Presto" name led to a rename: the project became **Trino** in December 2020, and the fork has since diverged substantially from Facebook's prestodb[^1]. Starburst is the primary commercial vendor and employs many core maintainers.

The defining tension: Trino is a query engine, not a database. It gives you one SQL dialect over heterogeneous sources and excellent read performance, but it inherits the durability, consistency, and metadata semantics of whatever it queries. It is also, by default, not fault-tolerant — a single lost worker fails the whole query — which shapes how it is operated in production.

## Getting Started

The fastest path is the official Docker image with the built-in TPC-H connector:

```bash
docker run -d --name trino -p 8080:8080 trinodb/trino
docker exec -it trino trino
```

Then query the sample catalog from the CLI prompt:

```sql
-- Federated query semantics: catalog.schema.table
SELECT
    n.name AS nation,
    count(*) AS customers
FROM tpch.tiny.customer c
JOIN tpch.tiny.nation n ON c.nationkey = n.nationkey
GROUP BY n.name
ORDER BY customers DESC;
```

Real deployments add a catalog per source. A catalog is a properties file naming a connector plus its config, e.g. `etc/catalog/iceberg.properties`:

```properties
connector.name=iceberg
iceberg.catalog.type=hive_metastore
hive.metastore.uri=thrift://metastore:9083
```

Building from source requires a modern JDK — Java 25 as of mid-2026[^2] — and Maven (`./mvnw clean install -DskipTests`).

## Architecture / How It Works

A Trino cluster is one **coordinator** and N **workers**. The coordinator parses SQL, plans and optimizes the query, then schedules pipelined tasks onto workers; workers pull data through connectors and stream intermediate results to each other over an exchange. Results stream back through the coordinator to the client.

Key internals:

- **Connector SPI.** Every data source is a plugin implementing the Service Provider Interface: metadata, splits, page sources, and increasingly *pushdown* (predicate, projection, aggregation, limit, join, TopN). Pushdown quality varies enormously by connector — a JDBC connector may push a filter into the remote database, while another materializes the whole scan into Trino. This is the single biggest determinant of real-world performance.
- **Execution model.** Queries become a tree of stages; stages split into tasks; tasks run pipelines of operators over columnar **pages** in memory. Execution is vectorized and pipelined, not batch-materialized between stages, which is why latency is low but memory pressure is high.
- **Cost-based optimizer.** Uses table statistics (row counts, NDV, histograms where available) to choose join order and distribution. Missing or stale stats degrade plans badly.
- **Fault-tolerant execution (FTE).** Introduced via "Project Tardigrade" in 2022, an opt-in mode that spools exchange data to an external store (S3, HDFS) so failed tasks can retry without restarting the whole query. It trades latency for resilience and is aimed at long ETL-style jobs rather than interactive dashboards[^3].

The lake-analytics connectors — **Hive, Iceberg, Delta Lake** — are where most Trino usage concentrates, reading Parquet/ORC directly from object storage and coordinating through a metastore (Hive Metastore or a REST/Glue catalog).

## Production Notes

The differentiators between "works in a demo" and "works at scale" are almost all operational:

- **Memory is the constant enemy.** Trino holds working sets in JVM heap. `query.max-memory` (cluster-wide per query) and `query.max-memory-per-node` cap it; exceed them and the query is killed, not slowed. Spilling to disk exists for some operators (joins, aggregations) but is slower and not universal. Sizing worker heap and GC (G1) is core tuning work.
- **Not fault-tolerant by default.** Without FTE, a single worker OOM, spot-instance reclaim, or network blip fails the entire query. Long batch jobs on cheap/preemptible infrastructure need FTE; interactive clusters usually accept the retry-the-query model.
- **Coordinator is a single point.** One coordinator per cluster plans and schedules everything; there is no built-in coordinator HA. Very high concurrency or very large plans make it the bottleneck.
- **The metastore is a shared dependency.** Hive Metastore latency, small-file explosion, and partition-count blowups on the lake side surface as Trino slowness. Iceberg's metadata layer mitigates some of this but adds its own maintenance (snapshot expiry, compaction).
- **Aggressive upgrade cadence.** Trino ships roughly monthly with a single incrementing version number and no long-term-support branch. Breaking changes (SPI changes, config renames, connector behavior, rising JDK floor) accumulate; the JDK requirement in particular has climbed from Java 11 to Java 25 in a few years[^2]. Skipping many releases at once is painful — most operators upgrade frequently in small steps.
- **Security is deployment-your-problem.** Authentication (LDAP, OAuth2, Kerberos), TLS, and fine-grained access control (file-based or Open Policy Agent / Ranger) are all configurable but off by default; a fresh cluster is wide open.

## When to Use / When Not

**Use when:**
- You need interactive SQL across a data lake (Iceberg/Delta/Hive on S3/GCS/ADLS) at scale.
- You want to federate queries over many heterogeneous sources with one dialect and one client.
- Read-heavy analytics and BI dashboards are the workload, and you can run dedicated compute.

**Avoid when:**
- You need a transactional database (OLTP), row-level updates, or strong consistency — Trino is read-optimized and inherits source semantics.
- Your data fits on one machine — DuckDB or ClickHouse will be simpler and faster.
- You want turnkey durability and fault tolerance without operating an MPP cluster and its memory tuning.
- You need a storage engine; Trino stores nothing and must sit on top of one.

## Alternatives

- prestodb/presto — the Facebook-lineage original; shares heritage and SQL but has diverged in code, connectors, and roadmap. Use it when you're already on the Presto/Meta/Velox stack.
- apache/spark — Spark SQL covers overlapping analytics but is batch/framework-oriented; use it when you need general-purpose data processing and ML pipelines, not just interactive SQL.
- ClickHouse/ClickHouse — a columnar OLAP database (owns its storage); use it for single-system, ultra-low-latency analytics rather than federated lake queries.
- apache/doris (or StarRocks/starrocks) — MPP OLAP databases with native storage and MySQL-protocol SQL; use when you want a self-contained warehouse, not a storage-less federator.
- duckdb/duckdb — in-process analytical SQL; use when the data is single-node scale and you want zero cluster operations.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013 | Presto open-sourced by Facebook. |
| — | 2018 | Original authors fork Facebook's Presto as PrestoSQL. |
| 351 | 2020-12 | First release under the new **Trino** name after the trademark rename[^1]. |
| ~393 | 2022 | Fault-tolerant execution (Project Tardigrade) introduced[^3]. |
| 449 | 2024 | Reproducible builds supported from this version onward[^4]. |
| current | 2026-07 | Actively maintained; near-monthly releases, Java 25 build/runtime floor[^2]. |

## References

[^1]: Trino Software Foundation, "We're rebranding PrestoSQL as Trino" — 2020-12-27. https://trino.io/blog/2020/12/27/announcing-trino.html
[^2]: Trino README, build requirements (Java 25.0.1+). https://github.com/trinodb/trino/blob/master/README.md
[^3]: Trino docs, "Fault-tolerant execution". https://trino.io/docs/current/admin/fault-tolerant-execution.html
[^4]: Trino README — reproducible builds supported as of version 449. https://github.com/trinodb/trino/blob/master/README.md

## Tags

java, sql, query-engine, distributed-systems, big-data, data-lake, analytics, iceberg, delta-lake, olap, presto, federation
