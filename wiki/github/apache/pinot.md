# apache/pinot

> A real-time distributed OLAP datastore built for low-latency, high-concurrency analytics on streaming and batch data.

[GitHub repo](https://github.com/apache/pinot) ·
[Official website](https://pinot.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/pinot/blob/master/LICENSE)

## Overview

Pinot is a columnar OLAP datastore designed for one specific shape of workload: aggregations and filters over large, largely-immutable event data, answered in tens of milliseconds, at query rates in the hundreds of thousands per second. It was built at LinkedIn (open-sourced 2015) to power user-facing analytics like "Who Viewed Your Profile" and later hardened at Uber[^1]. It entered the Apache Incubator in October 2018 and graduated to a Top-Level Project in August 2021[^2]. The commercial vendor behind most core development is StarTree, founded by Pinot's original authors.

The defining design choice is that Pinot optimizes for **serving-side latency and concurrency**, not for ingestion flexibility or ad-hoc analytical breadth. Data is stored as immutable columnar **segments**, heavily indexed at write time (inverted, range, StarTree pre-aggregation, text, JSON, geospatial, timestamp), so that read queries touch as little data as possible. The cost is that Pinot is not a general-purpose warehouse: mutations are limited, joins were historically absent, and the operational surface (ZooKeeper, Helix, segment lifecycle, deep store) is large.

The central tension in adopting Pinot is that it is unusually fast for the narrow band of workloads it targets — user-facing dashboards, real-time metrics, personalization — and unusually heavy to operate for anything outside that band. Teams that treat it as "another SQL database" tend to underestimate the cluster-management and tuning burden.

## Getting Started

The fastest path is the bundled QuickStart, which spins up a full cluster and loads sample tables:

```bash
docker run -p 9000:9000 apachepinot/pinot:latest QuickStart -type batch
# Query console + REST API at http://localhost:9000
```

Query via the SQL REST endpoint once tables are loaded:

```sql
-- Aggregation over a columnar table (single-stage engine)
SELECT playerName, SUM(homeRuns) AS hr
FROM baseballStats
WHERE yearID >= 2000
GROUP BY playerName
ORDER BY hr DESC
LIMIT 10
```

Building from source requires JDK 21+ for the services (clients/SPI still target Java 11 bytecode)[^3]:

```bash
git clone https://github.com/apache/pinot.git && cd pinot
./mvnw clean install -DskipTests -Pbin-dist -Pbuild-shaded-jar
```

## Architecture / How It Works

A Pinot cluster is four component types coordinated by **Apache Helix** on top of **Apache ZooKeeper**[^4]. Helix (also from LinkedIn) owns cluster state — which segment lives on which node, and in what state. This is a hard dependency, not an optional add-on.

- **Controller** — cluster brain. Manages metadata, segment assignment, table configs, and periodic tasks. Talks to ZooKeeper.
- **Broker** — query gateway. Parses SQL, prunes segments via metadata, scatters query fragments to the servers holding relevant segments, gathers and merges results.
- **Server** — stores segments and executes query fragments locally. Split logically into offline (batch) servers and real-time (streaming) servers, though a node can do both.
- **Minion** — runs background jobs: segment merge/rollup, purge (GDPR-style deletes), and real-time-to-offline segment conversion.

**Segments** are the unit of storage and distribution: a self-contained columnar chunk with its own indexes and dictionaries, persisted to a **deep store** (S3, HDFS, GCS, ADLS) as the source of truth and cached on servers for serving. Real-time tables consume from a stream (Kafka, Pulsar, Kinesis) into in-memory **consuming segments**, which are periodically flushed and committed to immutable segments once a row/time/size threshold is hit.

Query execution has two engines. The original **single-stage engine** is a scatter-gather design: brokers fan out to servers, each server aggregates locally, the broker merges. It is extremely fast but cannot do distributed joins — everything is single-table (dimension lookups aside). The **multi-stage query engine** (v2), introduced around 2022, adds intermediate stages with data shuffle between servers to support joins and window functions[^5]. It is more capable and more resource-intensive; the two engines coexist and are selected per query.

The **StarTree index** is Pinot's signature feature: a configurable materialized pre-aggregation tree that trades storage for turning many group-by queries into near-constant-time lookups.

## Production Notes

**ZooKeeper and Helix are load-bearing.** Cluster metadata churn (frequent segment commits, large tables, aggressive rebalances) puts real pressure on ZooKeeper. Undersizing or neglecting the ZK ensemble is a common outage cause. Treat it as critical infrastructure, not a sidecar.

**Real-time ingestion is memory-sensitive.** Consuming segments live in memory until they commit. Flush thresholds set too high cause OOMs under load; set too low they fragment tables into millions of tiny segments that hurt query pruning and balloon ZooKeeper metadata. Tuning `realtime.segment.flush.threshold.*` per table is unavoidable at scale.

**Upsert tables have hard constraints.** Upsert requires the stream to be partitioned by primary key, and all segments of a partition must colocate on one server (no cross-server primary-key resolution). The primary-key-to-location map is held per server and grows with key cardinality — high-cardinality upsert tables consume large, often off-heap, memory. This does not scale the way append-only tables do.

**Rebalance is an operation, not a button.** Adding/removing servers or changing replication triggers segment movement that must be run and monitored (`includeConsuming`, downtime vs. no-downtime modes). Poorly planned rebalances can degrade query latency or overwhelm the deep store.

**Indexing is a design decision, paid at write time.** StarTree and inverted indexes multiply segment size and build cost. Over-indexing inflates storage and ingestion latency; under-indexing surrenders Pinot's whole value proposition. There is no automatic index advisor — you model queries first.

**Multi-stage engine costs.** Joins via v2 shuffle data across servers and can consume far more memory/CPU than single-stage queries. It narrows but does not close the gap with join-first MPP systems; large joins remain a place where Pinot is not the strongest choice.

## When to Use / When Not

**Use when:**
- You need user-facing analytics: dashboards and in-product metrics queried directly by end users at high QPS with millisecond latency.
- Your data is append-heavy event/time-series data with many dimensions and metrics.
- You need real-time and batch data unified in one table with the same low-latency serving guarantees.
- Predictable filter/aggregation query patterns let you pre-model indexes and StarTree.

**Avoid when:**
- You want a general-purpose data warehouse for exploratory ad-hoc SQL and heavy multi-table joins — a warehouse or MPP engine fits better.
- Your workload is mutation-heavy (frequent updates/deletes beyond the upsert model).
- You lack the operational appetite for ZooKeeper + Helix + segment lifecycle management; a single-node columnar store is far simpler.
- Query patterns are unknown or constantly changing, making index pre-modeling impractical.

## Alternatives

- apache/druid — the closest architectural sibling; real-time OLAP with segment storage. Choose Druid if you prefer its operational model and roll-up ingestion; both target user-facing analytics.
- ClickHouse/ClickHouse — columnar OLAP with rich SQL and strong single-node performance. Use it for analyst-driven ad-hoc queries and joins rather than high-concurrency user-facing serving.
- StarRocks/starrocks (and apache/doris) — MPP analytical databases designed joins-first. Prefer these when distributed joins and warehouse-style queries are central.
- apache/kylin — pre-aggregation OLAP on Hadoop. Use when you live in the Hadoop ecosystem and want cube-based acceleration.
- questdb/questdb — time-series-first database. Use for simpler time-series ingest/query without Pinot's cluster complexity.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015 | Open-sourced by LinkedIn[^1]. |
| Incubation | 2018-10 | Enters Apache Incubator[^2]. |
| TLP | 2021-08 | Graduates to Apache Top-Level Project[^2]. |
| 0.11 | 2022 | Multi-stage query engine (joins) introduced[^5]. |
| 1.0.0 | 2023-09 | First 1.x release; multi-stage engine matured. |
| 1.x | 2024–2026 | Ongoing releases; JDK 21+ required to build services[^3]. |

## References

[^1]: Apache Pinot README and project site — origins at LinkedIn, adoption at Uber. https://pinot.apache.org/
[^2]: Apache Software Foundation, Pinot project incubation and graduation records. https://incubator.apache.org/projects/pinot.html
[^3]: Apache Pinot README, "Building Pinot" — JDK 21+ for services, Java 11 bytecode for clients/SPI. https://github.com/apache/pinot#building-pinot
[^4]: Pinot documentation, "Architecture" — Controller/Broker/Server/Minion on Helix + ZooKeeper. https://docs.pinot.apache.org/basics/architecture
[^5]: Pinot documentation, "Multi-stage query engine" — distributed joins and window functions. https://docs.pinot.apache.org/reference/multi-stage-engine

## Tags

java, olap, real-time-analytics, columnar-database, distributed-systems, apache, streaming, kafka, low-latency, data-store
