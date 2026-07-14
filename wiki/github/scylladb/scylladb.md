# scylladb/scylladb

> A Cassandra- and DynamoDB-compatible NoSQL database written in C++ on a shard-per-core runtime, trading the JVM for hand-tuned hardware saturation.

[GitHub repo](https://github.com/scylladb/scylladb) ·
[Official website](https://www.scylladb.com) ·
License: ScyllaDB Source Available (see relicensing note below)

## Overview

ScyllaDB is a wide-column NoSQL data store that speaks Apache Cassandra's CQL protocol and, optionally, Amazon DynamoDB's HTTP API (a subsystem called Alternator). It was started around 2014-2015 by Avi Kivity and Dor Laor — the same people behind the KVM hypervisor and OSv — under the company Cloudius Systems, later renamed ScyllaDB. The pitch has always been narrow and concrete: reimplement Cassandra's data model and wire protocol in C++ on a purpose-built asynchronous runtime, and thereby get an order of magnitude more throughput per node with no garbage-collector pauses[^1].

The defining engineering bet is the **Seastar** framework[^2]: a shared-nothing, thread-per-core (ScyllaDB calls it *shard-per-core*) runtime where each CPU core owns a slice of the data, its own memory, and its own network and disk queues, and cores communicate by explicit message passing rather than shared locks. This is what lets ScyllaDB avoid the JVM entirely and manage memory, I/O scheduling, and CPU scheduling itself instead of deferring to the OS page cache and a garbage collector. The cost is that ScyllaDB is opinionated about the machine it runs on — it wants whole cores, most of the RAM, and a fast local disk, and it performs worst when those assumptions are violated.

The defining *governance* tension is licensing. ScyllaDB historically shipped an AGPL-3.0 "Open Source" edition alongside a proprietary Enterprise edition. That split has been collapsed into a single product distributed under the **ScyllaDB Software License Agreement**, a source-available (not OSI-approved) license — the repository's license file is `LICENSE-ScyllaDB-Source-Available.md`, v1.1 dated April 2026[^3]. Read the terms before assuming this is open source in the sense Cassandra is; it is not.

## Getting Started

The fastest real path is the official Docker image:

```bash
docker run --name scylla -d scylladb/scylla --smp 1
docker exec -it scylla cqlsh
```

`--smp 1` pins it to a single shard, which is what you want on a laptop; in production you give it every core. Then, in `cqlsh`:

```sql
CREATE KEYSPACE catalog WITH replication =
  {'class': 'NetworkTopologyStrategy', 'replication_factor': 3};

CREATE TABLE catalog.products (
  id     uuid PRIMARY KEY,
  name   text,
  price  decimal
);

INSERT INTO catalog.products (id, name, price)
  VALUES (uuid(), 'widget', 9.99);
```

Any Cassandra CQL driver (Java, Python `cassandra-driver`, Go `gocql`, Rust `scylla`) connects unchanged; ScyllaDB ships shard-aware drivers that route queries to the owning core for extra throughput.

## Architecture / How It Works

The whole system is built on Seastar's core idea: **do not share mutable state between cores.**

- **Shard-per-core.** Data is hash-partitioned across cores within a node, then across nodes. A given partition is owned by exactly one shard on each replica. Reads and writes are routed to the owning shard; there are no cross-core locks on the hot path. This is why per-node throughput scales close to linearly with core count.
- **Own memory and cache.** ScyllaDB uses `O_DIRECT` and manages its own row-based unified cache rather than relying on the Linux page cache. Memory is pre-allocated per shard. There is no JVM heap and no GC — latency tails are set by I/O and scheduling, not stop-the-world pauses.
- **Userspace I/O scheduling.** Seastar runs its own disk I/O scheduler and priority classes, so background work (compaction, repair, streaming) can be throttled relative to foreground queries. This is the mechanism behind ScyllaDB's workload-prioritization story.
- **Storage engine.** Log-structured merge trees with SSTables, the same lineage as Cassandra, including the familiar compaction strategies (size-tiered, leveled, time-window) plus ScyllaDB's Incremental Compaction Strategy for reduced space amplification.
- **Raft for the control plane.** Recent versions moved schema and cluster topology onto a Raft consensus group, replacing the old gossip-plus-eventual-consistency approach to metadata. This makes concurrent schema changes and topology operations strongly consistent instead of a source of races[^4].
- **Tablets.** Data distribution has been moving from Cassandra-style vnodes to *tablets* — dynamically sized, individually schedulable partitions of the token range that can be split, merged, and migrated on the fly, enabling faster node addition and automatic load balancing[^4].
- **Alternator.** A DynamoDB-compatible API implemented on top of the same storage engine, disabled by default and enabled by config.

The coupling to hardware is the through-line: Seastar, the custom allocator, the I/O scheduler, and the shard model are one co-designed system, and its performance claims only hold when the OS is configured to get out of the way.

## Production Notes

- **It wants the whole machine.** ScyllaDB runs startup checks for CPU pinning, interrupt affinity, disk performance, and clocksource, and refuses non-optimal configurations unless you pass `--developer-mode 1`. Do **not** run production with developer mode on. `scylla_setup` / the provided tuning scripts exist for a reason.
- **Hot partitions punish the shard model.** Because a partition lives on one shard, a single hot partition saturates one core while others idle — you cannot spread it. Data modeling to avoid hot partitions and large partitions matters more here than on a shared-heap database.
- **Large partitions and large cells** cause latency spikes and memory pressure; ScyllaDB emits warnings above configurable thresholds. Model around them rather than tuning them away.
- **Filesystem and disk.** XFS on fast local NVMe is the supported, tested path; networked or throttled disks undercut the I/O scheduler's assumptions.
- **Repair and cleanup** are operational obligations, as with Cassandra: run scheduled repair (ScyllaDB Manager automates this) to maintain consistency, and cleanup after topology changes.
- **Licensing changes deployment math.** Under the source-available license, "Commercial Customer" and usage-restriction clauses apply; teams that adopted the old AGPL edition for its open-source guarantees should re-read the terms before upgrading past the relicensing boundary[^3]. This is a business decision, not just a version bump.
- **Cassandra compatibility is high but not total.** Most CQL, drivers, and tooling work unchanged, but some Cassandra features, virtual tables, and edge semantics differ; validate migrations against your actual query set rather than assuming drop-in parity.

## When to Use / When Not

**Use when:**
- You run Cassandra-shaped workloads (high write throughput, wide rows, tunable consistency) and want more per-node throughput with predictable p99 latency and no GC tuning.
- You have, or can provision, dedicated multi-core nodes with fast local disks.
- You want a DynamoDB-compatible API you can self-host.

**Avoid when:**
- You need an OSI-approved open-source license with the freedoms AGPL Cassandra provides — the relicensing changes that calculus.
- Your workload is small, bursty, or shares nodes with other services; ScyllaDB's advantages assume it owns the hardware.
- You need relational features, multi-row ACID transactions, or joins — this is a NoSQL wide-column store, not a SQL database.
- Your access pattern is dominated by unavoidable hot partitions.

## Alternatives

- apache/cassandra — the API and data model ScyllaDB clones; JVM-based, genuinely Apache-2.0 open source. Use it when the license or the mature ecosystem matters more than per-node throughput.
- cockroachdb/cockroach — distributed SQL with serializable transactions. Use it when you need relational semantics and strong consistency rather than a wide-column store.
- yugabyte/yugabyte-db — distributed SQL/CQL hybrid (Postgres + Cassandra APIs). Use it when you want CQL compatibility plus a Postgres surface.
- valkey/valkey (or redis) — in-memory key-value. Use it when your working set fits in RAM and you need microsecond latency, not durable big-data storage.
- Amazon DynamoDB — the other API ScyllaDB emulates. Use it when you want a fully managed service and are committed to AWS.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-12 | Repository created; Cloudius Systems / early ScyllaDB[^1]. |
| 1.0 | 2016-09 | First GA. CQL and Cassandra wire compatibility on Seastar. |
| 2.0 | 2017 | Continued CQL parity, performance work. |
| 3.0 | 2019 | Materialized views and secondary indexes, new SSTable format. |
| 4.0 | 2020 | Alternator (DynamoDB-compatible API), lightweight transactions. |
| 5.0 | 2022 | Raft introduced for the control plane (initially schema). |
| 6.0 | 2024 | Tablets and Raft-based strongly consistent topology[^4]. |
| 2025.x | 2025 | Unified year-versioned product; move to source-available licensing[^3]. |
| 2026.2 | 2026 | Current release line on the `master` branch[^5]. |

Version numbers and dates before the unified 2025.x line reflect the former Open Source edition and are approximate; confirm against release notes for a specific deployment.

## References

[^1]: ScyllaDB, "What is ScyllaDB?" and company history. https://www.scylladb.com/product/
[^2]: Seastar framework — shared-nothing, thread-per-core C++ runtime. https://seastar.io/ and https://github.com/scylladb/seastar
[^3]: ScyllaDB Software License Agreement v1.1, in-repo `LICENSE-ScyllaDB-Source-Available.md` (last updated 2026-04-12). https://github.com/scylladb/scylladb/blob/master/LICENSE-ScyllaDB-Source-Available.md
[^4]: ScyllaDB architecture docs — Raft and tablets. https://docs.scylladb.com/
[^5]: scylladb/scylladb tags (e.g. `scylla-2026.2.1`). https://github.com/scylladb/scylladb/tags
[^6]: Repository README and HACKING.md (frozen toolchain, C++23 build). https://github.com/scylladb/scylladb/blob/master/README.md

## Tags

cpp, nosql, database, wide-column, cassandra-compatible, dynamodb-compatible, seastar, shard-per-core, distributed-database, source-available, cql
