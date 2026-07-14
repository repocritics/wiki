# yugabyte/yugabyte-db

> A PostgreSQL-compatible distributed SQL database — Postgres's query layer bolted onto a Raft-replicated, auto-sharded storage engine.

[GitHub repo](https://github.com/yugabyte/yugabyte-db) ·
[Official website](https://www.yugabyte.com) ·
License: Apache-2.0 (core) / Polyform-Free-Trial-1.0.0 (Anywhere)

## Overview

YugabyteDB is a distributed SQL database that reuses the actual PostgreSQL
query layer — a fork of the PG C source — sitting on top of its own
distributed storage engine, DocDB[^1]. It targets cloud-native OLTP workloads
that need Postgres semantics plus horizontal scale, automatic failover, and
multi-region deployment: the "we outgrew a single Postgres box but don't want
to rewrite for a NoSQL store" case. The transaction design is explicitly
modeled on Google Spanner, using Raft consensus for replication and hybrid
logical clocks (HLC) for ordering[^2].

The company (Yugabyte, Inc., founded by ex-Facebook/Nutanix engineers) opened
the repo in 2017 and relicensed to 100% Apache 2.0 in 2019, moving formerly
enterprise features (distributed backups, encryption, CDC, read replicas) into
the open-source core[^3]. This is the key positioning against its closest
rival, cockroachdb/cockroach, which sits under the source-available BSL rather
than an OSI license.

The defining tension: YSQL is not "Postgres-like," it *is* Postgres's parser,
planner, and executor — but running against a distributed KV store instead of
the local heap. That buys real compatibility (extensions, stored procedures,
triggers, most of the type system) at the cost of a query layer that lagged
mainline Postgres badly. YugabyteDB was pinned to the PostgreSQL 11.2 fork for
years and only rebased to PostgreSQL 15 in the v2.25 preview (Jan 2025) and the
v2025.1 stable release (July 2025)[^4].

## Getting Started

Single-node local cluster via the `yugabyted` wrapper:

```bash
# Docker
docker run -d --name yugabyte -p 7000:7000 -p 9000:9000 \
  -p 5433:5433 -p 9042:9042 \
  yugabytedb/yugabyte:latest \
  bin/yugabyted start --daemon=false

# Connect with the bundled Postgres-compatible shell (ysqlsh)
docker exec -it yugabyte bin/ysqlsh -h yugabyte
```

YSQL speaks the Postgres wire protocol on port 5433 (not 5432), so `psql` and
any Postgres driver work directly:

```sql
CREATE TABLE users (
  id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL
) SPLIT INTO 6 TABLETS;         -- pre-split shards up front

INSERT INTO users (name) VALUES ('Tom'), ('Brad');
SELECT * FROM users;
```

The `SPLIT INTO` clause is YugabyteDB-specific DDL — a visible seam where the
distributed storage layer leaks through the Postgres surface.

## Architecture / How It Works

Two layers, deliberately decoupled:

1. **YQL query layer** — pluggable APIs. **YSQL** is the forked PostgreSQL
   query engine (the primary API). **YCQL** is a separate Cassandra-QL-like
   API with its own semantics[^1]. They share storage but not query
   compatibility; YSQL is where nearly all investment goes.
2. **DocDB** — the distributed document store. Built on a heavily customized
   fork of RocksDB (LSM-tree storage), it shards each table into **tablets**,
   replicates each tablet via its own **Raft** group (default RF=3), and
   applies distributed ACID transactions with two-phase commit coordinated
   through HLC timestamps[^2].

Two process types run the cluster: **yb-master** (metadata, tablet placement,
load balancing — not on the data path) and **yb-tserver** (hosts tablet peers,
serves reads/writes). Unlike Spanner, there is no TrueTime hardware; HLC
provides the ordering guarantee in software, which is the main correctness
subtlety operators should understand.

Notable internals:

- **Tablet splitting** — tablets auto-split as they grow, or can be pre-split
  with `SPLIT INTO`/`SPLIT AT`. Hot-shard and split behavior is a recurring
  tuning topic.
- **Colocation** — small tables can be colocated in a single tablet to avoid
  cross-node fan-out for join-heavy small schemas.
- **xCluster replication** — asynchronous replication between clusters
  (unidirectional or bidirectional) for two-region and DR topologies; this is
  eventual, not synchronous, consistency across the link.
- **Isolation levels** — Snapshot, Serializable, and Read Committed are
  supported; Read Committed became a default on new v2025.2 universes[^5].

The PostgreSQL fork is the architectural double-edged sword: it delivers
genuine compatibility, but every mainline PG advance (new planner features,
extensions, syntax) has to be pulled into the fork and reconciled with the
distributed executor. The Cost-Based Optimizer, bitmap scan, and parallel
query — all standard in Postgres — were only enabled by default on new
universes in the v2025.2 stable release[^5].

## Production Notes

**Latency is not single-node Postgres latency.** A cross-shard or multi-row
transaction pays Raft consensus plus 2PC round trips between nodes. Single-key
operations are fast; anything touching multiple tablets or nodes has a latency
floor set by network round trips inside the quorum. Teams migrating from a
single Postgres instance routinely find per-query latency higher and must
design schemas (colocation, primary-key sharding, pre-splitting) around data
locality rather than assuming it's free.

**PostgreSQL compatibility has a version lag and a feature gap.** Because YSQL
is a fork, extensions must be explicitly ported — you cannot assume an
arbitrary PG extension works. Behavior around sequences, `SERIAL`, and some
planner paths differs from mainline. The long PG 11.2 pin meant apps depending
on PG 12–15 features were blocked until 2025; verify your target version, not
"Postgres" in the abstract.

**Operational footprint is real.** You run and monitor yb-master and
yb-tserver processes, reason about tablet counts and placement, and size RF and
fault domains (zone/region/cloud). This is heavier than a managed single-node
Postgres or a read-replica setup. The core is predominantly C++ (GitHub labels
the repo C); builds are large.

**In-place upgrades have preconditions.** The v2025.1 online upgrade path
across the PG 15 rebase requires the source cluster to already be on
v2024.2.3.0 or later — you cannot jump arbitrary versions across the rebase
boundary[^4]. Plan upgrade chains, and test the downgrade path.

**License split — read LICENSE.md before you deploy the management plane.**
The core database (`src/`, `java/`) is Apache 2.0. **YugabyteDB Anywhere** (the
`managed/` self-hosted control plane) is under the Polyform Free Trial License
1.0.0, which is *not* an open-source license and restricts production use[^6].
The default build produces only the Apache-2.0 binaries; the operator UI you
may actually want carries different terms. The GitHub API reports the repo
license as `NOASSERTION` for exactly this reason.

**Issue volume.** The tracker carries thousands of open issues, typical for a
database of this surface area; treat it as a signal of breadth and active
development, not neglect — the repo is pushed to continuously.

## When to Use / When Not

**Use when:**
- You need genuine PostgreSQL compatibility (extensions, PL/pgSQL, drivers)
  *and* horizontal scale-out beyond one node.
- You require RPO=0 automatic failover across zones, regions, or clouds.
- You want a fully OSI-licensed (Apache 2.0) distributed SQL core, not a
  source-available BSL one.
- Your workload is cloud-native OLTP where correctness under failure matters
  more than absolute single-query latency.

**Avoid when:**
- A single Postgres node (or Postgres + read replicas) already meets your scale
  — you'd take on distributed-systems latency and ops cost for nothing.
- You need the very latest mainline PostgreSQL features or arbitrary extensions
  on day one; the fork lags and requires porting.
- Your workload is analytical/OLAP; this is an OLTP engine, not a columnar
  warehouse.
- You want a hands-off managed experience and won't run a fleet of
  master/tserver processes (consider the hosted YugabyteDB Aeon instead).

## Alternatives

- cockroachdb/cockroach — the closest competitor; Spanner-inspired distributed
  SQL with Postgres-wire compatibility, but a reimplemented (not forked) SQL
  layer and BSL source-available license. Choose it if you accept BSL terms and
  prefer its ecosystem; choose Yugabyte for a true PG fork and Apache 2.0 core.
- pingcap/tidb — distributed SQL that is MySQL-compatible (on TiKV storage).
  Use it when your app targets MySQL rather than Postgres.
- citusdata/citus — a Postgres *extension* that shards real, unforked Postgres.
  Use it when you want to scale out while staying on mainline Postgres and can
  live with its distribution model instead of a separate engine.
- vitessio/vitess — MySQL horizontal sharding (the YouTube/PlanetScale
  lineage). Use it for large MySQL fleets, not Postgres semantics.
- postgres/postgres — plain PostgreSQL. Use it when one node scales and you
  don't need multi-node consensus; it will be simpler and lower-latency.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-source | 2017-10 | Repository opened; distributed SQL over DocDB[^1]. |
| relicense | 2019-07 | Moved to 100% Apache 2.0; enterprise features open-sourced[^3]. |
| 2.0 | 2019-09 | First GA of the 2.x line. |
| date-based | ~2024 | Switched to calendar versioning (2024.x, 2025.x). |
| 2.25 (Preview) | 2025-01 | PostgreSQL fork rebased 11.2 → 15.0[^4]. |
| v2025.1 (Stable) | 2025-07 | First stable PG 15; online upgrade across the rebase[^4]. |
| v2025.2 (Stable) | 2025-12 | CBO, bitmap scan, parallel append, Read Committed default on new universes[^5]. |

## References

[^1]: YugabyteDB README, "What is YugabyteDB" and "Architecture". https://github.com/yugabyte/yugabyte-db
[^2]: YugabyteDB docs, "Transactions" (Spanner-based design, Raft, hybrid logical clocks). https://docs.yugabyte.com/stable/architecture/transactions/
[^3]: Yugabyte, "YugabyteDB is now 100% open source" — 2019. https://www.yugabyte.com/blog/
[^4]: YugabyteDB docs, "Release notes v2025.1" (PostgreSQL 11.2 → 15.0 rebase, upgrade preconditions). https://docs.yugabyte.com/stable/releases/ybdb-releases/v2025.1/
[^5]: YugabyteDB docs, "Release notes v2025.2" (features enabled by default on new universes). https://docs.yugabyte.com/stable/releases/ybdb-releases/v2025.2/
[^6]: YugabyteDB LICENSE.md (Apache 2.0 core vs Polyform Free Trial for Anywhere/`managed/`). https://github.com/yugabyte/yugabyte-db/blob/master/LICENSE.md

## Tags

distributed-sql, database, postgresql, cloud-native, raft, oltp, distributed-database, multi-region, c-plus-plus, scale-out, spanner
