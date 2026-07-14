# pingcap/tidb

> A distributed, MySQL-compatible SQL database with a separated compute/storage architecture and a bolted-on columnar engine for HTAP.

[GitHub repo](https://github.com/pingcap/tidb) ·
[Official website](https://www.tidb.io/) ·
[License: Apache-2.0](https://github.com/pingcap/tidb/blob/master/LICENSE)

## Overview

TiDB ("Ti" for Titanium) is an open-source NewSQL database built by PingCAP, first released around 2016[^1]. Its pitch is horizontal scalability with the MySQL wire protocol: applications talk to it as if it were MySQL 8.0, but data is sharded across a cluster and replicated with Raft rather than living on a single primary. It targets workloads that outgrow a single MySQL instance but where teams do not want to rewrite around a sharding middleware or a non-relational store.

The defining architectural choice is the split between the SQL layer (this repo, the `tidb-server`, written in Go) and the storage layer (TiKV, a separate Rust project, and its Placement Driver, PD). `tidb-server` is stateless: it parses SQL, plans queries, and pushes computation down to TiKV, but holds no data itself. This lets compute and storage scale independently, but it also means "TiDB" the product is really three or four coordinated services, and this repository is only one of them. Reading the code here without TiKV/PD context is misleading.

The second defining tension is HTAP. TiDB adds TiFlash, a columnar engine that replicates from TiKV via a Multi-Raft Learner, so the same cluster can serve OLTP row lookups and OLAP scans without an ETL pipeline[^2]. This is genuinely useful but doubles the operational surface. Note also that as of 2026 the repo's own description has been rewritten around "agentic workloads," vector search, and agent memory — a marketing repositioning toward the AI wave that outpaces what most of the mature codebase is actually about. Treat the transactional SQL engine as the real product; the agent framing is newer and thinner.

## Getting Started

The standard way to try TiDB is TiUP, PingCAP's cluster manager, which spins up a full playground (TiDB + TiKV + PD) locally:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://tiup.io/ | sh
tiup playground
```

Then connect with any MySQL client:

```bash
mysql --host 127.0.0.1 --port 4000 -u root
```

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_RANDOM,   -- AUTO_RANDOM avoids write hotspots, unlike AUTO_INCREMENT
  name VARCHAR(128),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name) VALUES ('Tom'), ('Brad');
SELECT * FROM users;
```

The default port is 4000, not MySQL's 3306. For production, deployment is via TiUP on VMs or TiDB Operator on Kubernetes; the `tiup playground` process is single-machine and not durable.

## Architecture / How It Works

TiDB the cluster is four components:

1. **`tidb-server`** (this repo, Go) — stateless SQL layer. Parses SQL, builds logical/physical plans, and executes them by issuing key-range requests to TiKV. Coprocessor pushdown ships filters and aggregations to the storage nodes so less data crosses the network.
2. **TiKV** (separate repo, Rust) — the distributed key-value store that actually holds data. Rows are encoded into KV pairs; the keyspace is split into ~96 MB **Regions**, each a Raft group with (by default) three replicas.
3. **PD (Placement Driver)** — the cluster brain. Allocates timestamps (the TSO, a global logical clock), tracks Region locations, and rebalances/schedules Regions across TiKV nodes.
4. **TiFlash** — optional columnar replica for analytics, kept in sync as a Raft learner.

Transactions use a Percolator-style two-phase commit coordinated through the PD timestamp oracle, giving snapshot-isolation semantics across nodes[^3]. Every transaction takes a start timestamp and a commit timestamp from PD, which is why PD sits on the critical path of every write and why PD latency matters more than its modest resource footprint suggests.

The SQL layer contains a cost-based optimizer with statistics, join reordering, and index selection. Because execution is distributed, the planner must decide what to push into TiKV coprocessors versus compute in `tidb-server`; getting this wrong (e.g. a plan that pulls millions of rows up to the SQL node) is the usual root cause of a "TiDB is slow" report. TiFlash-vs-TiKV engine selection is likewise a planner decision driven by statistics.

The MySQL compatibility is protocol- and syntax-level, not a fork of MySQL. There is no shared code with MySQL/InnoDB; TiDB reimplements the surface. Consequently behavior can diverge in edge cases (see below).

## Production Notes

**It is a cluster, not a database process.** The minimum sensible production topology is 3 PD + 3 TiKV + 2 TiDB nodes, and TiFlash adds more. You do not run "a TiDB"; you operate a distributed system with Raft, rebalancing, and a scheduler. Teams that approach it as "MySQL that scales" underestimate the operational weight. Below roughly the point where a single large MySQL/Postgres node is saturated, TiDB is usually the wrong tool.

**MySQL compatibility is high but not total.** It is not a drop-in for every app. Known divergences historically include: no support for stored procedures/triggers/events, foreign keys were unenforced for years (added later, still with caveats), differing behavior on some `SET` variables, and no savepoint semantics identical to InnoDB in all versions. Validate your ORM and migration tooling against your target TiDB version rather than assuming parity.

**Hotspots are the classic footgun.** Monotonic primary keys (`AUTO_INCREMENT`, timestamp-prefixed keys) concentrate writes on the Region holding the newest keyspace, so one TiKV node saturates while others idle. The fixes — `AUTO_RANDOM`, `SHARD_ROW_ID_BITS`, or hash-partitioned keys — must be designed in up front; retrofitting them onto a hot table is painful.

**Latency profile differs from single-node.** Because commits require a Raft majority and PD round-trips, single-row point-write latency is higher than local MySQL. TiDB wins on aggregate throughput and large scans, not on the latency of one small transaction. Latency-critical single-shard workloads may regress after migration.

**HTAP doubles the surface.** TiFlash is a second storage system to size, monitor, and keep replicated. It is powerful for analytical queries but is not free operationally, and stale/lagging TiFlash replicas produce confusing "why is this query slow / wrong-looking" incidents.

**Upgrades and resources.** TiKV is memory- and disk-IO-hungry; undersized nodes cause Region scheduling churn. Version upgrades are cluster-wide rolling operations managed by TiUP/Operator and want a maintenance mindset. Read the release notes for compatibility flags between minor versions — optimizer and default-variable changes have altered plan behavior across releases.

## When to Use / When Not

**Use when:**
- You have genuinely outgrown a single MySQL/Postgres node on writes or data volume and want to keep the MySQL protocol.
- You need horizontal scale with strong consistency and automatic failover, not eventual consistency.
- You want transactional and analytical queries on the same live data without an ETL pipeline (HTAP).
- You can staff the operational cost of a distributed cluster (or pay for TiDB Cloud to absorb it).

**Avoid when:**
- A single well-tuned MySQL/Postgres instance (with read replicas) still fits — you would take on distributed-systems complexity for no benefit.
- Your workload is latency-sensitive single-row OLTP where per-transaction round-trip cost matters more than aggregate throughput.
- You rely on MySQL features TiDB does not fully replicate (certain stored-procedure/trigger logic, exact InnoDB edge-case behavior).
- You want an embedded or single-binary database — TiDB is inherently multi-service.

## Alternatives

- cockroachdb/cockroach — Postgres-wire distributed SQL with a similar Raft/range design; use it instead when your ecosystem is Postgres, not MySQL (note its BSL, not Apache, licensing).
- vitessio/vitess — MySQL sharding middleware that keeps real MySQL underneath; use it when you want to scale existing MySQL with maximal compatibility rather than a reimplemented engine.
- yugabyte/yugabyte-db — distributed SQL with both Postgres and Cassandra APIs; use it when you want multi-API or a Postgres surface.
- planetscale (Vitess-based, commercial) — use when you want managed MySQL sharding without operating the control plane yourself.
- Plain mysql/mysql-server or postgres/postgres — use when a single node still suffices; do not adopt distributed SQL preemptively.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-09 | Repository created; project incubated at PingCAP[^1]. |
| 1.0 | 2017-10 | First GA release. |
| 2.0 | 2018-04 | Optimizer and stability overhaul. |
| 3.0 | 2019-06 | Pessimistic transactions, performance work. |
| 4.0 | 2020-05 | TiFlash / HTAP columnar engine GA[^2]. |
| 5.0 | 2021-04 | MPP mode for TiFlash, clustered indexes. |
| 6.x | 2022 | Introduction of LTS release cadence. |
| 7.x | 2023 | Resource control, TTL, further MySQL 8.0 alignment. |
| 8.x | 2024–2025 | Continued LTS line; vector search added for AI/embedding workloads. |

## References

[^1]: PingCAP, "TiDB" project and company background. https://www.pingcap.com/tidb/
[^2]: PingCAP, "HTAP demystified — TiKV row store + TiFlash column store." https://www.pingcap.com/blog/htap-demystified-defining-modern-data-architecture-tidb/
[^3]: TiDB architecture documentation (SQL layer, TiKV, PD, transactions). https://docs.pingcap.com/tidb/stable/tidb-architecture

## Tags

go, database, distributed-sql, newsql, mysql-compatible, htap, raft, cloud-native, distributed-transactions, columnar-storage, kubernetes
