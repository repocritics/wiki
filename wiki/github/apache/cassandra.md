# apache/cassandra

> Masterless, partitioned wide-column store built for linear write scalability and multi-datacenter availability — at the cost of query flexibility and operational simplicity.

[GitHub repo](https://github.com/apache/cassandra) ·
[Official website](https://cassandra.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/cassandra/blob/trunk/LICENSE.txt)

## Overview

Cassandra is a distributed database that organizes data into a partitioned row store: rows grouped into tables, each with a required primary key, spread across a cluster by consistent hashing. It originated at Facebook (Avinash Lakshman, Prashant Malik) to solve inbox search, was open-sourced in 2008, and became an Apache top-level project in 2010[^1]. Its design deliberately fuses two 2007-era papers: Amazon Dynamo's masterless replication and gossip-based membership, and Google Bigtable's column-family data model and log-structured storage[^2].

The defining property is that **every node is equal**. There is no primary, no config server, no single point of failure. Any node can coordinate any read or write, replication is peer-to-peer, and the cluster stays available for writes even when nodes (or a whole datacenter) are down. This buys extreme write throughput and operational survivability across regions. The price is paid at query time and at 3am: Cassandra is an AP-leaning system with *tunable* consistency, not a relational database. Joins, subqueries, aggregations across partitions, and ad-hoc queries are absent or actively discouraged. You model tables around the queries you will run, denormalize aggressively, and accept that changing an access pattern often means adding a new table.

Cassandra is chosen when write volume, geographic distribution, and always-on availability dominate, and when the query set is known in advance — time-series, event logging, messaging, IoT telemetry, user activity feeds, and product catalogs at companies like Apple, Netflix, and Instagram. It is the wrong tool for anything needing transactions across entities, strong consistency by default, or exploratory querying.

## Getting Started

Cassandra ships as a tarball, a Docker image, or via package managers; it requires a supported JDK[^3].

```bash
docker run --name cass -p 9042:9042 -d cassandra:5.0
# wait ~30s for the node to bootstrap, then open a CQL shell:
docker exec -it cass cqlsh
```

```sql
-- Model tables query-first: this table answers "messages by user, newest first"
CREATE KEYSPACE chat WITH replication =
  { 'class': 'NetworkTopologyStrategy', 'dc1': 3 };

USE chat;

CREATE TABLE messages_by_user (
  user_id   uuid,
  sent_at   timeuuid,
  body      text,
  PRIMARY KEY ((user_id), sent_at)      -- partition key + clustering column
) WITH CLUSTERING ORDER BY (sent_at DESC);

INSERT INTO messages_by_user (user_id, sent_at, body)
  VALUES (now(), now(), 'hello');

-- Fast: hits a single partition. Cross-partition scans require ALLOW FILTERING (avoid).
SELECT * FROM messages_by_user WHERE user_id = ? LIMIT 20;
```

The partition key (`user_id`) decides which node owns the row; clustering columns (`sent_at`) decide sort order *within* a partition. Get this wrong and the schema fights you forever.

## Architecture / How It Works

**Cluster topology.** Nodes form a ring; each owns ranges of a token space. Since 3.0, each physical node holds many **virtual nodes (vnodes)** to smooth token distribution and speed up rebalancing. Membership and node state propagate via a **gossip** protocol — no coordinator, no ZooKeeper.

**Writes** go to a **commit log** (durability) and an in-memory **memtable**. When the memtable fills, it flushes to an immutable on-disk **SSTable**. This is a log-structured merge tree: writes are append-only and never do read-before-write, which is why write throughput is so high. Deletes don't remove data — they write a **tombstone** marker. Background **compaction** merges SSTables, discards overwritten rows, and evicts tombstones past their grace period.

**Reads** may touch the memtable plus multiple SSTables, merged by timestamp, filtered through bloom filters and partition indexes. Reads are inherently more expensive than writes; heavy tombstone accumulation or too many SSTables (poor compaction) degrade read latency sharply.

**Replication and consistency.** A replication factor (RF) per keyspace and datacenter controls how many copies exist. Consistency is **tunable per query** via consistency levels — `ONE`, `LOCAL_QUORUM`, `QUORUM`, `ALL`, etc. `LOCAL_QUORUM` on both reads and writes with RF=3 is the standard production setting for strong-enough consistency within a DC. Divergent replicas are reconciled by **hinted handoff**, **read repair**, and periodic **anti-entropy repair** (Merkle-tree comparison via `nodetool repair`).

**Query layer.** CQL is "SQL minus joins and subqueries, plus collections." The legacy Thrift RPC interface was removed in 4.0[^4]. Strong-consistency single-partition transactions exist via **lightweight transactions (LWT)**, implemented with Paxos — correct but multi-round-trip and slow. General multi-partition ACID transactions are being added through the **Accord** consensus protocol (CEP-15), still landing as of the 5.x line.

**Compaction strategies** are a first-class tuning knob: SizeTiered (STCS, write-heavy default), Leveled (LCS, read-heavy), TimeWindow (TWCS, time-series/TTL data), and the newer Unified Compaction Strategy (UCS) in 5.0. Choosing the wrong one is a common source of runaway disk and I/O.

## Production Notes

**Repair is not optional.** Deleted data can resurrect as "zombies" if a node misses a tombstone and it gets garbage-collected elsewhere. You must run repair on every node within `gc_grace_seconds` (default 10 days). Manual `nodetool repair` is operationally heavy; most shops run [Cassandra Reaper](https://cassandra-reaper.io/) to schedule and throttle it.

**JVM GC dominates tail latency.** Cassandra is Java, and long garbage-collection pauses show up directly as p99 read spikes. G1GC with a carefully sized heap (often 8–31 GB to stay under compressed-oops) is standard; the storage engine pushes bulk data off-heap to reduce pressure. GC tuning is a recurring operational chore.

**Data modeling is the whole game.** The dominant failure modes are all schema mistakes: **large partitions** (keep well under ~100 MB / low hundreds of thousands of rows — an unbounded partition key eventually kills the owning node), **hot partitions** (skewed key = one node saturated while others idle), and **tombstone floods** from queue-like delete patterns or collection overwrites (reads scanning past thousands of tombstones time out). TWCS + TTL is the correct pattern for expiring time-series; row-level deletes at high volume is an anti-pattern.

**Features to avoid.** **Materialized views** have shipped with known consistency edge cases and are effectively considered experimental — maintain denormalized tables by hand instead. **Secondary indexes** (the older 2i) scale poorly and fan out to every node; the 5.0 **Storage-Attached Indexes (SAI)** are the modern replacement and are far better, but querying off the partition key is still not free[^5]. `ALLOW FILTERING` in a query is a red flag, not a feature.

**Operational surface.** Bootstrapping, decommissioning, and token streaming move large amounts of data and must be done one node at a time. 4.0 added zero-copy streaming to make this faster. Multi-DC replication (`NetworkTopologyStrategy`) is a core strength but requires disciplined consistency-level choices (`LOCAL_QUORUM`, not `QUORUM`, to avoid cross-region latency). Counters and lightweight transactions have their own consistency caveats and should be used sparingly.

**Upgrades** are version-sensitive: you generally cannot skip major versions, SSTable formats change between them (requiring `nodetool upgradesstables`), and mixed-version clusters are only safe transiently during a rolling upgrade.

## When to Use / When Not

**Use when:**
- Write throughput and always-on availability across multiple datacenters/regions are the priority.
- The access patterns are known up front and map to partition-keyed lookups (time-series, event logs, messaging, feeds, IoT, catalogs).
- You can staff the operational load (repair, compaction, GC, capacity planning) or pay for a managed offering.

**Avoid when:**
- You need cross-entity transactions, joins, or ad-hoc analytical queries — reach for a relational or distributed-SQL database.
- Your dataset is small enough for a single PostgreSQL/MySQL node; Cassandra's overhead only pays off at scale.
- The team wants strong consistency by default or can't invest in careful data modeling.
- Your workload is read-heavy with unpredictable query shapes.

## Alternatives

- scylladb/scylladb — C++ reimplementation of Cassandra's protocol/data model; lower and more predictable latency, no JVM GC tuning. Use when you want Cassandra semantics with less operational pain and are willing to bet on a single vendor's core.
- apache/hbase — Bigtable-style wide-column store on HDFS with a strongly-consistent, master-based design. Use when you're already in the Hadoop ecosystem and want CP over AP.
- cockroachdb/cockroach — distributed SQL with real ACID transactions and strong consistency. Use when you need transactions and SQL, not just scale-out writes.
- yugabyte/yugabyte-db — distributed SQL that also exposes a Cassandra-compatible (YCQL) API. Use when you want a migration path off CQL toward transactions.
- mongodb/mongo — document store with far gentler operations and flexible querying. Use when developer velocity and ad-hoc queries matter more than write-scale extremes.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2008-07 | Released by Facebook; Apache Incubator 2009[^1]. |
| 1.0 | 2011-10 | First stable major; compression, improved read performance. |
| 2.0 | 2013-09 | CQL3 matures; lightweight transactions (Paxos), triggers. |
| 2.1 | 2014-09 | User-defined types, better counters. |
| 3.0 | 2015-11 | Storage-engine rewrite; materialized views; vnodes maturing. |
| 4.0 | 2021-07 | Thrift removed; audit logging, virtual tables, zero-copy streaming[^4]. |
| 4.1 | 2022-12 | Guardrails, pluggable auth, paxos v2 groundwork. |
| 5.0 | 2024-09 | Storage-Attached Indexes (SAI), vector search / ANN, trie memtables, UCS[^5]. |

## References

[^1]: Apache Cassandra history and Facebook origin. https://cassandra.apache.org/_/cassandra-basics.html
[^2]: Lakshman & Malik, "Cassandra — A Decentralized Structured Storage System," 2009 — Dynamo + Bigtable lineage. https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf
[^3]: Getting Started / installation and supported JDKs. https://cassandra.apache.org/doc/latest/cassandra/getting-started/
[^4]: Apache Cassandra 4.0 release (Thrift removal, zero-copy streaming, virtual tables). https://cassandra.apache.org/_/blog/Apache-Cassandra-4.0-is-Here.html
[^5]: Apache Cassandra 5.0 features — Storage-Attached Indexes and vector search. https://cassandra.apache.org/doc/latest/cassandra/getting-started/vector-search-quickstart.html

## Tags

database, distributed-database, nosql, wide-column-store, java, cql, high-availability, scalability, apache, lsm-tree, time-series, eventual-consistency
