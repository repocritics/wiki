# apache/hbase

> Distributed, versioned, wide-column store on top of Hadoop HDFS — an open-source implementation of Google's Bigtable.

[GitHub repo](https://github.com/apache/hbase) ·
[Official website](https://hbase.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/hbase/blob/master/LICENSE.txt)

## Overview

HBase is a distributed key-value / wide-column store modeled directly on Google's 2006 Bigtable paper[^1], built on top of Apache Hadoop and HDFS. It gives you a sorted, sparse, multidimensional map: rows are keyed by an arbitrary byte-string row key, columns are grouped into column families, and every cell is versioned by timestamp. It is designed for random, low-latency read/write access to very large tables (billions of rows, millions of columns) where a single-node database would not fit. It started inside the Hadoop project around 2007 (inspired by Powerset's Bigtable clone) and became a top-level Apache project in 2010.

The defining tension is that HBase optimizes for **linear scalability of random access over sorted row keys**, and almost every operational problem traces back to that choice. Data is physically partitioned into contiguous key ranges ("regions"); reads and writes for a given key always hit exactly one region server. This makes point lookups and short range scans fast and predictable, but it also means a poorly designed row key (monotonic timestamps, sequential IDs) funnels all traffic to one server — the "hotspotting" problem that dominates HBase design reviews. There are no native secondary indexes, no cross-row transactions beyond a single row, and no SQL in the core; you model your access patterns into the row key up front.

HBase is strongly consistent (CP in CAP terms) for single-row operations and does not use tunable quorum reads like Cassandra. That consistency, plus deep Hadoop integration (bulk load from MapReduce/Spark, HDFS as the storage layer), is why it persists in large-scale analytics and operational stacks despite heavy operational weight.

## Getting Started

HBase can run standalone (single JVM, local filesystem) for evaluation, or fully distributed over HDFS + ZooKeeper.

```bash
# Download a binary release, then start a standalone instance
wget https://dlcdn.apache.org/hbase/<version>/hbase-<version>-bin.tar.gz
tar xzf hbase-<version>-bin.tar.gz
cd hbase-<version>
bin/start-hbase.sh
bin/hbase shell
```

```
# In the HBase shell — a table with one column family
create 'users', 'info'

# Cells are (row key, family:qualifier, timestamp) -> value
put 'users', 'user#1001', 'info:name', 'Tom'
put 'users', 'user#1001', 'info:email', 'tom@example.com'

get 'users', 'user#1001'
scan 'users', { STARTROW => 'user#1000', STOPROW => 'user#2000' }
```

The primary programmatic interface is the Java client (`Connection`, `Table`, `Put`, `Get`, `Scan`). Non-JVM clients go through the Thrift or REST gateways, and Apache Phoenix adds a JDBC/SQL layer on top.

## Architecture / How It Works

HBase is a layered system that leans on other Apache projects rather than reimplementing them:

- **HMaster** — handles region assignment, DDL (create/alter/drop table), load balancing, and region-server failover. It is not on the read/write data path; a cluster keeps serving data while the master is briefly down, but cannot reassign regions or run admin ops.
- **RegionServers** — the workhorses. Each hosts some set of regions and handles all client reads and writes for those key ranges.
- **Regions** — a contiguous, sorted range of row keys for one table. Regions split automatically as they grow and are the unit of distribution and load balancing.
- **ZooKeeper** — coordination: liveness of region servers, location of the `hbase:meta` catalog, master election. Historically a hard dependency for cluster state.
- **HDFS** — durable storage. HBase writes its files (WAL, HFiles) to HDFS and inherits its replication and durability.

Storage is a **log-structured merge tree (LSM)**. A write is appended to the write-ahead log (WAL) on HDFS for durability, then applied to an in-memory `MemStore` per column family. When the MemStore fills, it flushes to an immutable, sorted **HFile** on HDFS. Reads must therefore merge results across the MemStore plus every HFile for a region, using Bloom filters and block indexes to skip files. **Compactions** periodically merge HFiles: minor compactions merge a few small files; major compactions rewrite all files for a region into one and physically drop deleted/expired cells (tombstones and TTL). This buys fast writes at the cost of read amplification and background I/O.

The read path uses a **BlockCache** (on-heap LRU, or off-heap BucketCache for large caches to dodge JVM GC) plus the OS page cache under HDFS. HBase 2.0 (2018) reworked much of the internals: a redesigned assignment manager built on a Procedure framework (AMv2), off-heap read/write paths, and an in-memory compacting MemStore[^2]. **Coprocessors** (observer and endpoint) let you run server-side code for triggers, secondary-index maintenance, or aggregation — powerful and also the most common way to destabilize a cluster.

The long-running architectural project is reducing the ZooKeeper dependency and moving cluster state into HBase's own catalog tables; this is a central theme of the 3.x line still in development at time of writing, so treat specifics as unsettled.

## Production Notes

**Row-key design is the whole game.** Sequential or timestamp-prefixed keys create write hotspots on a single region server. Standard mitigations are salting (prefixing with a hash bucket), field-swapping (e.g. reversed domain names), or hashing the key — each trades scan locality for write distribution. This decision is effectively irreversible without a full rewrite.

**Compaction and GC are the operational pain.** Major compactions rewrite entire regions and can saturate disk and network; most operators disable automatic major compaction and schedule it off-peak. Large JVM heaps for BlockCache/MemStore invite long GC pauses that can trip ZooKeeper session timeouts and cause a region server to be declared dead — hence the push toward off-heap BucketCache and careful heap sizing.

**Failure recovery is not instant.** When a region server dies, its regions are reassigned and each must replay its share of the WAL before serving reads again (MTTR is measured in seconds to minutes depending on WAL size and split throughput). During that window the affected key ranges are unavailable — availability is per-region, not cluster-wide.

**No native cross-row transactions or secondary indexes.** Atomicity is guaranteed only within a single row. Secondary indexes, joins, and SQL come from Apache Phoenix, which maintains index tables via coprocessors — adding its own operational surface and consistency caveats.

**Operational weight is high.** A real deployment means running and tuning HDFS, ZooKeeper, and HBase together, plus monitoring region counts, request skew, compaction queues, and store-file counts. Full-table scans and OLAP-style aggregation are anti-patterns; HBase is for keyed access, and analytics typically pair it with Spark/MapReduce bulk reads or export to a columnar engine.

**Upgrades are non-trivial.** The 1.x → 2.x jump changed the assignment manager, file layout details, and client APIs enough to require planning and testing; rolling upgrades across major versions have historically been the sharp edge. Pin client and server versions deliberately.

## When to Use / When Not

**Use when:**
- You need random, strongly-consistent single-row read/write access at a scale beyond one machine.
- Your access patterns are known and can be encoded into a sorted row key (time series with good bucketing, entity-by-key lookups, message/event stores).
- You are already running Hadoop/HDFS and want tight bulk-load integration with MapReduce or Spark.
- You need cell-level versioning and TTL out of the box.

**Avoid when:**
- You need ad-hoc queries, joins, secondary indexes, or SQL as first-class features (without adopting Phoenix).
- Your workload is analytical / full-scan heavy — a columnar analytics store fits better.
- You want multi-datacenter active-active writes with tunable consistency — Cassandra's model is a better fit.
- You lack the operational capacity to run HDFS + ZooKeeper + HBase; a managed store removes most of this burden.
- Your dataset is small enough for a single relational database.

## Alternatives

- apache/cassandra — use instead when you want a masterless, highly-available (AP) design with tunable consistency and easy multi-DC writes, and no HDFS/ZooKeeper stack.
- apache/accumulo — use instead when you need the same Bigtable model plus cell-level security labels (originated at the NSA).
- scylladb/scylladb — use instead when you want Cassandra's data model with C++/shard-per-core latency and lower operational tuning overhead.
- apache/kudu — use instead when you need fast columnar scans for analytics alongside fast row updates, without the LSM read-amplification.
- Google Cloud Bigtable (managed, not OSS) — use instead when you want HBase's model and a compatible client without operating the cluster yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2007 | Started as a Hadoop contrib module, inspired by Bigtable. |
| TLP | 2010-05 | Became a top-level Apache project[^1]. |
| 0.96 | 2013-10 | "Singularity" release; protobuf-based wire format, metrics overhaul. |
| 1.0 | 2015-02 | First 1.x GA; stabilized client API[^3]. |
| 2.0 | 2018-04 | AMv2 assignment, Procedure framework, off-heap paths, in-memory compaction[^2]. |
| 2.x | 2018– | Long-lived stable line; ongoing point releases. |
| 3.0 | in dev | Reducing ZooKeeper reliance, catalog-driven cluster state (unreleased at time of writing). |

## References

[^1]: Apache HBase project home and reference guide. https://hbase.apache.org/book.html
[^2]: Apache HBase 2.0.0 release notes / "What's New in HBase 2.0". https://hbase.apache.org/2.0/
[^3]: Apache HBase downloads and release archive. https://hbase.apache.org/downloads/

## Tags

java, database, nosql, wide-column-store, bigtable, hadoop, hdfs, distributed-systems, lsm-tree, big-data, apache
