# cockroachdb/cockroach

> A distributed SQL database with PostgreSQL wire compatibility, built on a replicated key-value store — survives node, rack, and datacenter loss without manual failover.

[GitHub repo](https://github.com/cockroachdb/cockroach) ·
[Official website](https://www.cockroachlabs.com) ·
License: CockroachDB Software License (source-available, not OSI-approved)[^1]

## Overview

CockroachDB is a horizontally-scalable, strongly-consistent SQL database written in Go, started in 2014 by ex-Google engineers (Spencer Kimball, Peter Mattis, Ben Darnell) and modeled on the design ideas in Google's Spanner paper[^2]. The pitch is a single logical SQL database that shards and replicates itself across many nodes, keeps serving through machine and datacenter failures, and speaks the PostgreSQL wire protocol so existing drivers and ORMs mostly work unchanged. The name is the marketing thesis: hard to kill.

The defining architectural bet is that **SQL is a thin layer on top of a transactional, geo-distributed key-value store**. Data is range-partitioned, each range is Raft-replicated (default 3 replicas), and distributed ACID transactions run at `SERIALIZABLE` isolation by default over MVCC. Unlike Spanner, it does not require atomic clocks — it uses hybrid logical clocks plus a bounded clock-skew assumption, which is both its cleverest idea and its sharpest operational edge[^3].

The other defining fact about CockroachDB in 2026 is its license. It is **not open source in the OSI sense**. The self-hosted "core" was Apache 2.0 until 2019, moved to the Business Source License (BSL 1.1) that year, and in November 2024 (v24.3 and later) moved again to the unified CockroachDB Software License (CSL), which is free only for organizations under a revenue threshold and otherwise requires a paid enterprise license[^1]. GitHub reports the license as `NOASSERTION` because it maps to no standard SPDX identifier. Evaluate it as a commercial product with published source, not as a community-governed OSS project.

## Getting Started

```bash
# Single-node insecure dev instance (do NOT use in production)
cockroach start-single-node --insecure --listen-addr=localhost:26257

# Open the built-in SQL shell
cockroach sql --insecure --host=localhost:26257
```

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- avoid monotonic PKs
  email STRING UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO users (email) VALUES ('tom@example.com');
SELECT id, email FROM users;
```

Application code connects with any PostgreSQL driver:

```go
// database/sql + lib/pq or pgx — Postgres wire protocol
db, _ := sql.Open("postgres", "postgresql://root@localhost:26257/defaultdb?sslmode=disable")
```

## Architecture / How It Works

The system is a layered stack, each layer building on the one below[^4]:

1. **SQL** — parser, cost-based optimizer, and a distributed execution engine (DistSQL) that pushes computation to the nodes holding the data.
2. **Transactional KV** — distributed, serializable MVCC transactions. Uses a two-phase-commit variant with a transaction record and write intents; contention is resolved by pushing/aborting conflicting transactions.
3. **Distribution** — the keyspace is split into **ranges** (default target size 512 MiB, historically 64 MiB), which split and merge automatically and rebalance across nodes based on load and capacity.
4. **Replication** — each range is an independent Raft group. A write commits once a quorum of replicas acknowledges. The Raft leader (leaseholder) serves reads locally.
5. **Storage** — a per-node LSM engine. The original RocksDB (C++) backend was replaced by **Pebble**, a Go-native LSM store, which became the default in v20.2 (2020)[^5].

**Clocks.** There is no TrueTime. Nodes use hybrid logical clocks and assume a maximum clock offset (default 500 ms). To preserve serializability the timestamp of a read may be pushed forward within an "uncertainty interval," which can force transaction restarts. A node that detects its clock has drifted beyond the max offset **deliberately crashes itself** to avoid violating consistency[^3].

The SQL-on-KV layering is the source of both the horizontal scalability and a class of surprising performance characteristics: a query's cost is often dominated not by SQL execution but by how many ranges and Raft groups it touches and where their leaseholders live.

## Production Notes

**Clock sync is mandatory, not optional.** Run NTP/chrony (or a cloud time service) on every node. Sustained skew beyond `--max-offset` self-terminates nodes; this is the single most common cause of unexpected node loss in badly-provisioned clusters[^3].

**Sequential keys create hotspots.** A monotonically increasing primary key (a `SERIAL`/sequence, `ORDER`-style bigint, or timestamp) funnels every insert into the same range and therefore the same leaseholder node — a write hotspot that no amount of adding nodes will fix. Use `UUID` keys or `gen_random_uuid()`, or hash-sharded indexes (`USING HASH`) for time-series-shaped data. This is the number-one CockroachDB footgun.

**Serializable means you must handle retries.** Because the default isolation is `SERIALIZABLE`, contended transactions abort with retry errors (Postgres error code `40001`). Every application must implement a transaction retry loop; code ported straight from single-node Postgres without one will throw under load. `READ COMMITTED` isolation was added in later releases (v23.2+) to reduce this, but it changes correctness guarantees and must be chosen deliberately.

**Cross-region writes pay quorum latency.** In a multi-region deployment a write must reach a Raft quorum. If replicas span continents, single-row write latency is bounded below by inter-region round-trip time. Multi-region features (regional-by-row tables, follower reads, locality-aware leaseholder placement) exist precisely to manage this, but they require deliberate schema and topology design — you do not get low cross-region write latency for free.

**Disk and IO.** It is an LSM store: it wants fast local NVMe/SSDs and is sensitive to disk stalls. Undersized disks show up as compaction backlog and read amplification. Admission control (added to shed load under overload) helps but does not substitute for adequate IOPS.

**Upgrades are rolling but finalize.** Major-version upgrades are done node-by-node, then a cluster-version "finalization" step enables new features and is **not reversible** — once finalized you cannot downgrade to the prior major version. Test the upgrade path in staging and understand the auto-finalization behavior before you start.

**Not drop-in Postgres.** Wire compatibility is real, but the schema-change, isolation, and performance model differ. Assume every nontrivial Postgres app needs review, not just a connection-string swap.

## When to Use / When Not

**Use when:**
- You need a single SQL database that survives zone/region failure with automatic failover and no read-replica promotion dance.
- You need horizontal write scaling with strong consistency (serializable), not eventual consistency.
- You have data-residency/geo-partitioning requirements (keep EU rows in the EU) expressible at the row/table level.
- Your team already speaks Postgres and wants to keep the dialect and drivers.

**Avoid when:**
- A single well-provisioned Postgres instance (plus read replicas) covers your load — you'd be paying distributed-systems latency and operational cost for scale you don't need.
- Your workload is analytical/OLAP with huge scans — CockroachDB is OLTP-first; pair it with a warehouse instead.
- Your access pattern relies on monotonic keys or high single-row contention and can't be reshaped.
- You require an OSI-approved open-source license or want to avoid a revenue-gated commercial license[^1].

## Alternatives

- yugabyte/yugabyte-db — distributed SQL with Postgres compatibility under Apache 2.0; use it when a genuinely open-source license is a hard requirement.
- pingcap/tidb — MySQL-compatible distributed SQL with a separate HTAP/analytics engine (TiFlash); use it when your stack is MySQL and you also want column-store analytics.
- vitessio/vitess — horizontal sharding for MySQL (powers PlanetScale); use it to scale an existing MySQL app rather than adopt a new SQL engine.
- Google Cloud Spanner — the fully-managed system CockroachDB is modeled on; use it when you're on GCP, want zero cluster ops, and accept full vendor lock-in.
- postgres/postgres — single-node relational workhorse; use it when you don't need multi-node survival or horizontal write scale, which is most applications.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-02 | Repository created; project public[^2]. |
| 1.0 | 2017-05 | First production (GA) release. |
| 2.0 | 2018-04 | Distributed SQL, JSONB, performance work. |
| 19.1 | 2019-04 | Calendar versioning begins; core relicensed to BSL 1.1[^1]. |
| 20.2 | 2020-11 | Pebble becomes the default storage engine[^5]. |
| 21.1 | 2021-05 | Larger default range size; multi-region SQL primitives. |
| 22.x | 2022 | Multi-region GA maturity, admission control for overload. |
| 23.2 | 2024 | `READ COMMITTED` isolation introduced. |
| 24.3 | 2024-11 | Unified CockroachDB Software License (CSL) takes effect[^1]. |

## References

[^1]: CockroachDB README, "Licensing"; all versions v24.3+ (and later patch series) are under the CockroachDB Software License. https://github.com/cockroachdb/cockroach/blob/master/LICENSE
[^2]: Cockroach Labs, company/project background. https://www.cockroachlabs.com/
[^3]: Cockroach Labs, "Living Without Atomic Clocks" — clock offset, hybrid logical clocks, and node self-termination on skew. https://www.cockroachlabs.com/blog/living-without-atomic-clocks/
[^4]: CockroachDB Architecture Overview (SQL → transactional KV → distribution → replication → storage). https://www.cockroachlabs.com/docs/stable/architecture/overview.html
[^5]: Cockroach Labs, "Pebble: A RocksDB-inspired key-value store written in Go." https://www.cockroachlabs.com/blog/pebble-rocksdb-kv-store/

## Tags

go, distributed-database, sql, newsql, postgresql-compatible, raft-consensus, oltp, source-available, multi-region, mvcc
