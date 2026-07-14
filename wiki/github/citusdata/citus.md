# citusdata/citus

> A PostgreSQL extension that shards tables across a cluster of Postgres nodes — distributed scale-out without leaving Postgres.

[GitHub repo](https://github.com/citusdata/citus) ·
[Official website](https://www.citusdata.com) ·
[License: AGPL-3.0](https://github.com/citusdata/citus/blob/main/LICENSE)

## Overview

Citus is a PostgreSQL extension — not a fork — that turns a single Postgres server into the coordinator of a distributed cluster[^1]. It shards tables across worker nodes, routes and parallelizes queries, and presents the cluster to applications as an ordinary Postgres database over the standard wire protocol. Because it loads via `shared_preload_libraries` and hooks the planner and executor rather than rewriting Postgres, it tracks upstream releases closely; Citus 13.0 (February 2025) supports Postgres 17[^2].

The project began at Citus Data (founded 2011). Early versions were a Postgres fork; the codebase was rewritten as a pure extension around Citus 5.0 in 2016. Microsoft acquired Citus Data in January 2019[^3] and now runs it as the engine behind Azure Cosmos DB for PostgreSQL (formerly "Hyperscale (Citus)"). For several years the shard rebalancer and other "enterprise" features were closed-source; Citus 11 (2022) open-sourced everything under AGPL-3.0[^4], which is the current state of the repo.

The defining tradeoff is the **distribution column**. You pick one column per table to hash-shard on, and Citus's entire performance model follows from that choice: queries that filter on it route to a single shard, joins between tables co-located on it run locally, and everything else fans out across the cluster or is disallowed. Getting the distribution column right (usually `tenant_id` for SaaS, `device_id`/`user_id` for time-series) is the whole game, and changing it later means redistributing data.

## Getting Started

Try a single-node cluster in Docker:

```bash
docker run -d --name citus -p 5500:5432 -e POSTGRES_PASSWORD=pw citusdata/citus
docker exec -it citus psql -U postgres
```

Or add the extension to an existing Postgres install — set `shared_preload_libraries = 'citus'` in `postgresql.conf`, restart, then:

```sql
CREATE EXTENSION citus;

CREATE TABLE events (
  device_id  bigint,
  event_id   bigserial,
  event_time timestamptz default now(),
  data       jsonb not null,
  PRIMARY KEY (device_id, event_id)   -- must include the distribution column
);

-- hash-shard the table on device_id
SELECT create_distributed_table('events', 'device_id');
```

Queries filtering on `device_id` are routed to one shard; aggregates without it are parallelized across all shards.

## Architecture / How It Works

A Citus cluster is one **coordinator** node plus N **worker** nodes. All are ordinary Postgres processes running the extension.

- **Coordinator** — holds the distributed metadata (which shards live where) in catalog tables, and hooks the Postgres planner/executor. It rewrites an incoming query into per-shard SQL, dispatches those tasks to workers, and merges results. Application connections normally target the coordinator.
- **Workers** — store shards as regular Postgres tables named like `events_102008`. A worker has no idea it's part of a distributed system; it just runs the SQL the coordinator sends.
- **Distributed tables** — hash-sharded on the distribution column into a fixed shard count (default 32). Tables sharded on the same column type are **co-located**, so joins and foreign keys on that column execute entirely within a worker with no cross-node traffic.
- **Reference tables** — replicated in full to every node, for dimension tables joined on non-distribution columns and for foreign keys from distributed tables.
- **Columnar storage** — a table access method (`USING columnar`) that compresses data and speeds analytical scans; usable independently of sharding.

The **adaptive executor** decides per query whether to route to a single shard, run a real-time parallel scatter-gather, or perform a repartition join (reshuffling data between workers) when a join isn't on the distribution column. Distributed transactions use two-phase commit, and there is a distributed deadlock detector. Since Citus 11, workers also carry the metadata, so any node can serve distributed queries ("query from any node")[^4] — this relieves the coordinator as the sole query entry point, though it remains the DDL and metadata authority.

## Production Notes

- **The distribution column is a one-way door.** Choosing poorly (or picking a low-cardinality / skewed column) creates hot shards and query fan-out. Changing it requires `undistribute_table` + re-`create_distributed_table`, i.e. moving the data.
- **Constraints must include the distribution column.** Primary keys and unique constraints on a distributed table must contain the distribution column; there are no cluster-wide unique indexes on arbitrary columns. Foreign keys between distributed tables require co-location and inclusion of the distribution column. Applications built on global uniqueness need rework.
- **Cross-shard queries have real cost.** Anything that doesn't prune to a single shard opens connections to many shards; high concurrency of fan-out queries can exhaust worker connection slots. `citus.max_adaptive_executor_pool_size` and connection pooling (PgBouncer) matter at scale.
- **Rebalancing moves data.** Adding workers doesn't spread existing data until you run `rebalance_table_shards()`. The rebalancer is online (uses logical replication) but is I/O heavy; schedule it deliberately.
- **Coordinator failover is your job.** Citus doesn't ship built-in HA; you pair it with Postgres streaming replication + a failover tool (Patroni, or the managed Azure service which handles it for you).
- **Upgrades are two-dimensional.** You upgrade the Citus extension (`ALTER EXTENSION citus UPDATE`) and, separately, the Postgres major version — and all nodes must stay in step. Read the version-specific upgrade notes; skipping intermediate Citus majors is not always supported.
- **Not every Postgres feature distributes.** Some DDL, triggers, and extensions behave differently or need explicit propagation across the cluster; test the specific features you depend on rather than assuming transparency.

## When to Use / When Not

**Use when:**
- You have a multi-tenant SaaS app with a natural `tenant_id` — the canonical Citus fit, where nearly all queries prune to one shard.
- You've outgrown one Postgres node on CPU/memory/IO and want to scale out while keeping SQL, joins, foreign keys, and the Postgres extension ecosystem (PostGIS, etc.).
- You run real-time analytics over time-series/IoT data and want parallel aggregation plus columnar compression.
- You want horizontal scale without migrating off Postgres or rewriting your application.

**Avoid when:**
- Your workload has no clean distribution column and is dominated by cross-shard joins or global unique constraints.
- You need a leaderless, self-healing distributed SQL database with built-in HA and no coordinator — a purpose-built system fits better.
- A single large Postgres node (plus read replicas and native partitioning) already meets your needs — sharding adds operational complexity you don't want prematurely.
- You need strict low-latency single-row transactions that span shards.

## Alternatives

- cockroachdb/cockroach — use when you want a ground-up distributed SQL database with built-in HA and no coordinator, and can accept partial Postgres compatibility instead of the real extension ecosystem.
- yugabyte/yugabyte-db — use when you want Postgres wire/SQL compatibility distributed by default across nodes rather than bolted onto a single coordinator.
- pingcap/tidb — use when you want a horizontally scalable HTAP database with MySQL rather than Postgres compatibility.
- vitessio/vitess — use when your stack is MySQL and you need proven sharding at scale.
- timescale/timescaledb — use when your scaling problem is time-series volume on essentially a single Postgres node, not multi-node horizontal sharding.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011 | Citus Data founded; early CitusDB was a Postgres fork. |
| 5.0 | 2016-03 | Relaunched as a pure PostgreSQL extension[^1]. |
| — | 2019-01 | Microsoft acquires Citus Data[^3]. |
| 10 | 2021-03 | Columnar storage, single-node Citus, shard rebalancer open-sourced[^5]. |
| 11 | 2022 | Fully open source (AGPL-3.0); query from any node[^4]. |
| 12 | 2023 | Schema-based sharding (distributed schemas). |
| 13.0 | 2025-02 | PostgreSQL 17 support[^2]. |

## References

[^1]: "What it means to be a PostgreSQL extension" — Citus Data blog. https://www.citusdata.com/blog/2017/10/25/what-it-means-to-be-a-postgresql-extension/
[^2]: "Distribute PostgreSQL 17 with Citus 13.0" — 2025-02-06. https://www.citusdata.com/blog/2025/02/06/distribute-postgresql-17-with-citus-13/
[^3]: Microsoft, "Microsoft acquires Citus Data" — 2019-01-24. https://news.microsoft.com/2019/01/24/microsoft-acquires-citus-data-re-affirming-its-commitment-to-open-source-and-accelerating-azure-postgresql-performance-and-scale/
[^4]: "Citus 11 for Postgres goes fully open source" — 2022. https://www.citusdata.com/blog/2022/06/17/citus-11-goes-fully-open-source/
[^5]: "Citus 10: Columnar for Postgres, rebalancer, single-node, & more" — 2021-03-12. https://www.citusdata.com/blog/2021/03/12/citus-10-columnar-compression-single-node-shard-rebalancer/
[^6]: Citus SIGMOD '21 paper, "Citus: Distributed PostgreSQL for Data-Intensive Applications." https://doi.org/10.1145/3448016.3457551

## Tags

c, postgresql, distributed-database, sharding, multi-tenant, columnar-storage, scale-out, sql, database-extension, real-time-analytics
