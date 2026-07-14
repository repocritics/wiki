# prestodb/presto

> Distributed MPP SQL query engine that federates queries across Hive, Iceberg, relational, and object stores without moving the data.

[GitHub repo](https://github.com/prestodb/presto) ·
[Official website](http://prestodb.io) ·
[License: Apache-2.0](https://github.com/prestodb/presto/blob/master/LICENSE)

## Overview

Presto is a distributed SQL query engine built at Facebook in 2012 and open-sourced in 2013[^1]. It is designed for interactive analytics over large datasets: a coordinator parses and plans a query, then a fleet of workers executes it in memory, streaming intermediate results between stages rather than materializing them to disk. Its defining trait is federation — a single `SELECT` can join a Hive table, a MySQL table, and an Iceberg table because each data source is exposed through a pluggable connector. Presto is compute only; it owns no storage and no catalog, relying on an external metastore (typically Hive Metastore or AWS Glue).

The most important fact about this repository is the 2019 fork. The three original creators (Martin Traverso, Dain Sundstrom, David Phillips) left Facebook and forked the project as PrestoSQL, later renamed Trino in December 2020[^2]. `prestodb/presto` is the branch that stayed with Facebook/Meta and is now governed by the Presto Foundation under the Linux Foundation[^3]. `trinodb/trino` is the branch that followed the original authors. The two share a common ancestor and much of the same SQL semantics and connector concepts, but have diverged substantially in code, feature set, and release cadence. Choosing "Presto" today means choosing which of these two lineages you mean — a recurring source of documentation and community confusion.

Presto's second defining tension is memory. It was built for fast, in-memory interactive queries, not long-running batch ETL. A query that exceeds cluster memory historically fails rather than spilling gracefully, and a single worker crash fails the whole query. Much of the modern engineering effort — spilling, native C++ execution — exists to soften these limits.

## Getting Started

Presto ships as a tarball (coordinator + workers) plus a CLI. A minimal single-node setup:

```bash
# Download and unpack a release from prestodb.io/download
tar -xzvf presto-server-*.tar.gz
cd presto-server-*

# etc/config.properties (single-node acts as both coordinator and worker)
#   coordinator=true
#   node-scheduler.include-coordinator=true
#   http-server.http.port=8080
#   query.max-memory=5GB
#   discovery-server.enabled=true
#   discovery.uri=http://localhost:8080

bin/launcher run
```

```sql
-- Connect with the CLI and inspect the cluster
-- ./presto --server localhost:8080 --catalog hive --schema default
SELECT * FROM system.runtime.nodes;

-- Federated join: object-store table + relational table in one query
SELECT o.id, u.email
FROM hive.default.orders o
JOIN mysql.app.users u ON o.user_id = u.id;
```

Building from source requires Java 17, Maven 3.6.3+, and `./mvnw clean install`[^4]. Java 17 needs a long list of `--add-opens` reflective-access flags for certain catalogs.

## Architecture / How It Works

Presto runs a **coordinator/worker** topology. The coordinator receives SQL, parses it, builds and optimizes a distributed query plan, splits it into stages and tasks, and schedules those tasks onto workers. Workers pull data from connectors, run operators (scans, joins, aggregations), and exchange rows with each other over HTTP. There is no persistent state on workers between queries — the cluster is a stateless compute layer over external data.

Data access is entirely **connector-based**. A connector implements metadata (schemas, tables, columns), data location (splits), and page sources (actual row batches). The catalog/schema/table namespace maps `catalog.schema.table` onto a configured connector instance. Hive, Iceberg, Delta, MySQL, PostgreSQL, Kafka, Elasticsearch, and dozens of others exist. Predicate and projection pushdown into the connector is what makes federation practical rather than a full table scan.

The engine is a **staged, pipelined, in-memory MPP executor**. Intermediate results stream between stages instead of being written to disk (in the classic Java engine), which is why it is fast for interactive work and fragile for oversized joins. Spill-to-disk exists for some operators but is not a general fault-tolerance mechanism.

The largest current architectural investment is **Presto native (Prestissimo)** — a C++ rewrite of the worker built on Velox, Meta's C++ vectorized execution library[^5]. The coordinator remains Java; workers can be swapped for the native binary to gain vectorized, SIMD-friendly execution and lower memory overhead. This is a long-running effort and not every function or connector is covered by the native path.

## Production Notes

- **Memory is the operational ceiling.** `query.max-memory` (per-query, cluster-wide) and `query.max-memory-per-node` gate execution. Large joins/aggregations that exceed these fail with exceeded-memory errors. Sizing these against actual RAM, and enabling spill selectively, is the core tuning task. Presto is not the tool for terabyte-shuffle ETL.
- **No mid-query fault tolerance in the classic engine.** A single worker OOM or crash kills the whole query; you re-run it. Long-running batch jobs are exposed to this. (Fault-tolerant execution is far more mature on the Trino side — a real reason teams pick Trino for ETL.)
- **JVM tuning matters.** G1GC configuration, heap region size, and large-page settings materially affect stability under load. The engine is sensitive to GC pauses on the coordinator.
- **The coordinator is a single point of scheduling.** It plans every query and tracks every task; very high concurrency or huge plans pressure it. Multiple coordinators are not a standard HA story in the way workers scale.
- **Version scheme and upgrades.** prestodb releases on a `0.x` line (e.g. `0.28x`), distinct from Trino's integer versions (400+). Java baseline moved from 8 to 11 to 17 over time; the Java 17 jump requires the `--add-opens` flags and can break older deployment scripts. Connector behavior and SQL edge cases can shift between minor releases.
- **You must run a metastore.** Hive Metastore or Glue is a hard dependency for the common Hive/Iceberg catalogs; it is a separate service to operate, secure, and back up.
- **Security is bring-your-own.** Authentication (LDAP, Kerberos, JWT, OAuth2), TLS, and file/system access control are all configured, not defaulted. An unsecured coordinator is an open query endpoint to all connected data sources.

## When to Use / When Not

**Use when:**
- You need interactive, ad-hoc SQL over a data lake (Hive/Iceberg on S3/HDFS) with second-to-minute latency.
- You must query across heterogeneous sources (object store + RDBMS + streaming) in one SQL statement.
- You want to separate compute from storage and scale workers elastically.
- Your workload is analytical reads, not transactional writes.

**Avoid when:**
- You need reliable long-running batch ETL with mid-query fault tolerance — Spark or Trino's fault-tolerant execution fit better.
- You need a transactional (OLTP) database — Presto is read-analytics, not a system of record.
- You want a single-node embedded analytics engine — DuckDB or ClickHouse are far simpler.
- You want the lineage with faster feature velocity and are willing to switch names — evaluate Trino first.

## Alternatives

- trinodb/trino — the 2019 fork by Presto's original creators; generally faster feature cadence and stronger fault-tolerant execution. Use instead when you want the same query-federation model with more active batch/ETL support.
- apache/spark — use instead when your workload is heavy batch ETL or you need mid-job fault tolerance and a broad DataFrame/ML ecosystem, not interactive SQL latency.
- ClickHouse/ClickHouse — use instead when you own the storage and want a single-system columnar OLAP database with extreme scan speed rather than a federated query layer.
- duckdb/duckdb — use instead for embedded, single-node analytics on files without running a cluster.
- apache/doris — use instead when you want an MPP analytical database with built-in storage and real-time upserts rather than compute-only federation.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012 | Created at Facebook to replace Hive for interactive analytics[^1]. |
| — | 2013-11 | Open-sourced under Apache 2.0[^1]. |
| — | 2019-01 | Original creators fork the project as PrestoSQL[^2]. |
| — | 2019-09 | Presto Foundation formed under the Linux Foundation[^3]. |
| — | 2020-12 | PrestoSQL renamed Trino, formalizing the two-lineage split[^2]. |
| — | ongoing | Presto native (Prestissimo) C++ workers on Velox[^5]. |
| — | recent | Java 17 becomes the build/runtime baseline[^4]. |

## References

[^1]: Facebook Engineering, "Presto: Interacting with petabytes of data at Facebook" — 2013-11-06. https://engineering.fb.com/2013/11/06/core-infra/presto-interacting-with-petabytes-of-data-at-facebook/
[^2]: Trino, "Trino: The renaming of PrestoSQL" — 2020-12-27. https://trino.io/blog/2020/12/27/announcing-trino.html
[^3]: Linux Foundation / Presto Foundation. https://prestodb.io/foundation/
[^4]: prestodb/presto README — build requirements (Java 17, Maven 3.6.3+). https://github.com/prestodb/presto/blob/master/README.md
[^5]: Presto native execution and Velox. https://github.com/prestodb/presto/tree/master/presto-native-execution — https://github.com/facebookincubator/velox

## Tags

sql, distributed-systems, query-engine, big-data, mpp, data-lake, java, olap, hive, iceberg, analytics
