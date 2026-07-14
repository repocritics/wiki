# vitessio/vitess

> A database clustering system that puts a MySQL-compatible sharding and connection-pooling layer in front of many MySQL instances.

[GitHub repo](https://github.com/vitessio/vitess) ·
[Official website](https://vitess.io) ·
[License: Apache-2.0](https://github.com/vitessio/vitess/blob/main/LICENSE)

## Overview

Vitess is a horizontal-scaling layer for MySQL. It began at YouTube around 2011 to keep MySQL viable as query and data volume grew, was open-sourced by Google, and became a CNCF graduated project in 2019[^1]. Applications talk to a stateless proxy (VTGate) that speaks the MySQL wire protocol, so most clients connect as if to a single large MySQL server; behind it, data is spread ("sharded") across many real MySQL instances that Vitess coordinates.

The defining idea is **generalized sharding via a VSchema**. You declare how each table is partitioned (a "vindex" maps a column value to a shard), and Vitess routes, scatters, and re-aggregates queries accordingly. It also owns the operational hard parts that usually stop teams from sharding by hand: online resharding with a few-second atomic cutover, primary failover, connection pooling, and schema migration. Companies including Slack, Square/Block, GitHub, JD.com, and PlanetScale (a managed-Vitess vendor) run it at large scale[^2].

The central tension is that Vitess is not a drop-in MySQL. It is a distributed system with several server roles, a topology store, and a query planner that supports a large but incomplete subset of MySQL semantics. Cross-shard transactions, some joins, and certain SQL constructs behave differently or are unsupported. You adopt Vitess when a single MySQL genuinely no longer fits — not for convenience — and you accept operating a cluster in exchange.

## Getting Started

The fastest path is the local "examples" cluster or the Kubernetes operator. Locally, with a MySQL install present:

```bash
# macOS/Linux — install the vitess binaries
brew install vitess          # or build from source: make build

# bring up a local sharded example (uses the 101_initial_cluster scripts)
git clone https://github.com/vitessio/vitess
cd vitess/examples/local
./101_initial_cluster.sh
```

Once up, VTGate speaks the MySQL protocol, so you connect with any MySQL client:

```bash
mysql -h 127.0.0.1 -P 15306   # default VTGate MySQL port in the example
```

```sql
-- VTGate looks like MySQL; the VSchema decides which shard(s) run this
SELECT * FROM commerce.customer WHERE customer_id = 42;
```

## Architecture / How It Works

Vitess is several cooperating processes, not one server:

- **VTGate** — stateless proxy that clients connect to. It parses SQL, consults the VSchema, plans the query, routes it to the right shard(s), scatters/gathers across shards, and merges results. Horizontally scalable and where connection pooling to the backends happens.
- **VTTablet** — a sidecar in front of *each* MySQL (`mysqld`) instance. It manages the connection pool, query rewriting, row-limit and query guards, health checks, backups, and schema/replication operations for that one MySQL.
- **Topology service** — an external consistent store (etcd, ZooKeeper, or Consul) that holds cluster metadata: keyspaces, shards, which tablet is the current primary, and the VSchema. It is the source of truth VTGate and tooling read from.
- **vtctld / vtctldclient** — the admin control plane and CLI for cluster operations (resharding, reparenting, migrations).
- **VTOrc** — automated failure detection and repair; promotes a replica when a primary dies and fixes broken replication.

**Sharding model.** A *keyspace* is a logical database; it is split into *shards*, each backed by its own MySQL primary plus replicas. The **VSchema** maps tables to a sharding scheme using *vindexes* — the default is a hash vindex over a chosen column. Queries whose predicate hits the vindex column are routed to a single shard; queries without it become scatter queries across all shards.

**Reads and replication.** VTGate can route reads to replicas (with configurable freshness) and writes to primaries. Within Vitess, **VReplication** is the workhorse for moving and copying data between shards. It powers **MoveTables** (relocating tables between keyspaces) and **Reshard** (splitting or merging shards) as online operations: it copies existing rows, tails the binlog to catch up, then performs an atomic **cutover** that flips traffic in a few seconds. **VStream** exposes the same change-data stream to applications as a CDC feed.

**Query planning.** VTGate's planner (historically "V3", now the "Gen4" planner) decides whether a statement is single-shard, scatter, or needs cross-shard coordination. Cross-shard transactions default to *best-effort* commit; an opt-in *two-phase commit (2PC)* mode exists but carries latency and complexity costs and is used selectively.

## Production Notes

**It is a distributed system, priced accordingly.** The minimum real deployment is VTGate(s), a VTTablet per MySQL, a topology store, and control-plane tooling — plus the MySQL instances themselves. Running this outside Kubernetes is possible but most production users adopt the operator or a managed provider rather than hand-rolling process supervision and failover.

**MySQL compatibility is a subset, and the gaps are where migrations stall.** The planner supports a large fraction of MySQL SQL, but not all of it. Common friction: cross-shard joins that can't be pushed down become scatter-and-merge (or are rejected), some subqueries and correlated queries are unsupported on sharded keyspaces, `LAST_INSERT_ID` semantics differ, and cross-shard transactions are not fully ACID unless you enable 2PC. Test your actual query workload against VTGate before committing — the failures are query-shaped, not load-shaped.

**Scatter queries are the classic footgun.** A query missing the vindex column fans out to every shard. This works, hides the cost in development at small shard counts, and then degrades sharply as you add shards. Auditing for unintended scatters (VTGate exposes query-plan and per-query metrics) is standard operational hygiene.

**Resharding is online but not free.** MoveTables/Reshard copy data via VReplication, which adds load to source primaries and takes real wall-clock time proportional to data size. The cutover is fast, but the copy/catch-up phase must be planned around traffic, and you monitor VReplication lag throughout.

**Schema changes** run through Vitess's managed OnlineDDL (backed by strategies including its own vitess/`vreplication` executor and support for gh-ost/pt-osc style flows), which serializes and tracks migrations across shards rather than each shard drifting independently.

**Version cadence and upgrades.** Vitess ships roughly quarterly major releases with a documented deprecation/upgrade-order policy: within a cluster you generally upgrade VTTablet and VTGate in a supported order, and skipping multiple major versions is discouraged. Read the release notes — behavior changes in the planner and defaults land between majors.

## When to Use / When Not

**Use when:**
- A single MySQL (even a large one) can no longer hold your write volume or dataset, and you need horizontal sharding without rewriting the app to be shard-aware.
- You want online resharding, managed failover, and connection pooling as built-ins rather than bespoke tooling.
- You already run MySQL and want to keep the MySQL wire protocol and operational familiarity while scaling out.
- You're on Kubernetes and can use the operator or a managed Vitess service.

**Avoid when:**
- One MySQL (or a read-replica setup / Aurora-style managed MySQL) still comfortably fits — the operational overhead of Vitess is not worth it below that threshold.
- Your workload leans on full cross-shard ACID transactions or the complete MySQL SQL surface; the compatibility gaps will bite.
- You want a single-binary "just a database" — Vitess is a cluster with multiple roles and an external topology store.
- You aren't on MySQL/MySQL-compatible storage; Vitess is MySQL-specific.

## Alternatives

- planetscale/planetscale — managed, commercial Vitess; use when you want the sharding model without operating the cluster.
- cockroachdb/cockroach — distributed SQL (Postgres wire, not MySQL) with native cross-shard transactions; use when you want horizontal scale with full distributed ACID and no MySQL requirement.
- pingcap/tidb — MySQL-compatible distributed SQL with an integrated storage engine; use when you want MySQL wire compat but a purpose-built distributed store instead of many MySQLs.
- yugabyte/yugabyte-db — distributed SQL (Postgres-compatible) for geo-distributed workloads; use when Postgres semantics and multi-region are priorities.
- Amazon Aurora / Vitess-free vertical scaling — use when a single managed MySQL still fits and you only need read scale-out and HA.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011 | Started at YouTube to scale MySQL[^1]. |
| Open source | 2013 | Repository opened by Google (repo created 2013-06-27). |
| CNCF incubation | 2018 | Accepted into the CNCF. |
| Graduated | 2019-11 | Became a CNCF graduated project[^1]. |
| Gen4 planner | ~2021 | New query planner path added alongside V3. |
| v14–v22 | 2022–2026 | Roughly quarterly majors; planner, VReplication, and OnlineDDL maturation[^3]. |

## References

[^1]: CNCF, "Vitess" project profile and graduation announcement. https://www.cncf.io/projects/vitess/
[^2]: Vitess adopters list. https://github.com/vitessio/vitess/blob/main/ADOPTERS.md
[^3]: Vitess releases and release notes. https://github.com/vitessio/vitess/releases

## Tags

go, database, mysql, sharding, distributed-systems, horizontal-scaling, cncf, kubernetes, database-cluster, proxy, cloud-native, sql
