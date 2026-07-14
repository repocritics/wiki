# tikv/tikv

> Distributed, transactional key-value store in Rust — the storage layer under TiDB, promoted to a standalone Raft/RocksDB database.

[GitHub repo](https://github.com/tikv/tikv) ·
[Official website](https://tikv.org) ·
[License: Apache-2.0](https://github.com/tikv/tikv/blob/master/LICENSE)

## Overview

TiKV is a horizontally scalable key-value database that provides both raw KV APIs and ACID transactional APIs. It was created by PingCAP in 2016 as the storage engine for TiDB — their MySQL-compatible distributed SQL database — and later split out as a general-purpose store usable on its own[^1]. Its design borrows directly from Google's papers: BigTable for the range-partitioned data model, Spanner for externally consistent transactions, and Percolator for the two-phase commit transaction protocol layered over a KV store[^2].

The system is a CNCF graduated project (graduated 2020), one of the few databases at that tier and the flagship Rust database in the foundation[^3]. Consistency comes from Raft (one Raft group per data range, called a Region), and durable local storage comes from RocksDB. A separate control-plane service, the Placement Driver (PD), owns cluster metadata, timestamp allocation, and automatic rebalancing.

The defining tension: TiKV is technically standalone, but in practice most of its gravity is inside the TiDB ecosystem. Running it "bare" (raw or transactional client, no TiDB) is supported and real, but the documentation, tooling, tuning knowledge, and client maturity are heavily weighted toward the TiDB path. You get a serious distributed database, but you inherit an operational surface — PD, RocksDB, Raft, multi-Region rebalancing — that is closer to running a database cluster than a KV cache.

## Getting Started

The minimal deployment is one PD instance plus one TiKV instance. For anything beyond a smoke test, use TiUP (PingCAP's cluster manager) or the Docker Compose setup in the repo (3 PD + 3 TiKV).

```bash
# Minimal single-node: start PD, then TiKV pointed at it
./pd-server --name=pd --data-dir=/tmp/pd \
  --client-urls="http://127.0.0.1:2379"
./tikv-server --pd-endpoints="127.0.0.1:2379" \
  --addr="127.0.0.1:20160" --data-dir=/tmp/tikv
```

```python
# Raw KV via the Python client (no transactions)
from tikv_client import RawClient

client = RawClient.connect(["127.0.0.1:2379"])  # PD endpoint, not TiKV
client.put(b"foo", b"bar")
print(client.get(b"foo"))  # b'bar'
```

Clients always connect to PD, not directly to TiKV nodes — PD tells the client which Region lives on which store. The most production-ready client is Go (`tikv/client-go`); Java and Rust exist, and the Python client is best treated as experimental[^4].

## Architecture / How It Works

The data plane is organized around four nested concepts:

- **Node** — a physical machine running one or more Stores.
- **Store** — a single RocksDB instance on local disk.
- **Region** — the unit of data movement: a contiguous key range (default target ~96 MiB). Regions split and merge as data grows and shrinks.
- **Raft group** — each Region is replicated (typically 3 replicas) across Stores on different Nodes; the replicas form one Raft group with a leader that serves reads and writes.

**PD (Placement Driver)** is the brain. It allocates the globally unique, monotonic timestamps (TSO) that order transactions, tracks where every Region replica lives, and continuously issues scheduling commands to balance leaders and data across the cluster. PD is itself a small Raft-replicated cluster (usually 3 nodes) — it is a hard dependency, not optional. Lose quorum in PD and the cluster loses its ability to allocate timestamps and reschedule.

**Transactions** use a Percolator-style optimistic (and, since later versions, pessimistic) two-phase commit. A transaction picks a start timestamp from PD, buffers writes, then on commit picks a commit timestamp and runs prewrite + commit phases, using one key's lock as the "primary lock" to make the commit atomic. This gives snapshot isolation and externally consistent reads. The cost is that lock contention and the TSO round-trip to PD are real latency contributors under write-heavy, hot-key workloads.

**Storage** is RocksDB, and TiKV inherits RocksDB's LSM-tree behavior wholesale: write amplification, compaction stalls, and the need to tune block cache and compaction settings. There is also a **Coprocessor** framework (HBase-inspired) that pushes filters and aggregations down to the storage node so TiDB doesn't have to ship whole rows back — most of the coprocessor logic exists to serve TiDB's SQL executor.

The through-line: TiKV's correctness story (Raft + RocksDB + Percolator) is well-trodden and grounded in published designs, but every layer is an operational component you must understand. There is no way to run it as an opaque box.

## Production Notes

- **PD is a single point of coordination.** It is Raft-replicated so it tolerates node loss, but it is on the critical path for timestamp allocation. TSO latency and PD leader stability directly shape transaction throughput. Treat PD as a first-class cluster, not an afterthought.
- **RocksDB tuning is your problem.** Write amplification, compaction I/O, and block-cache sizing surface as tail-latency spikes. Fast local NVMe is effectively required; running TiKV on network block storage or spinning disks is a known way to get bad and unpredictable latency.
- **Hot Regions are the classic footgun.** Monotonic keys (timestamps, auto-increment IDs) concentrate writes on one Region leader, so one node saturates while the rest idle. PD's hot-Region scheduling and split logic mitigate this, but schema/key design that spreads writes is the real fix.
- **Rebalancing is not free.** When you add or remove nodes, PD moves Regions to rebalance, which consumes disk and network I/O and can compete with foreground traffic. Scaling events should be planned, and PD's scheduling limits tuned, rather than done blindly during peak load.
- **Version coupling with TiDB and PD.** TiKV, PD, and TiDB have compatible-version matrices; upgrades are expected to move roughly in lockstep and follow a rolling order (PD → TiKV → TiDB). Mixing arbitrary versions is not supported. Use TiUP or TiDB Operator for upgrades rather than hand-rolling them.
- **Standalone TiKV is a narrower path.** If you use raw/transactional KV without TiDB, you get less documentation, fewer battle-tested client features, and a smaller community answering questions. It works, but you are off the main trail.

## When to Use / When Not

**Use when:**
- You need a horizontally scalable KV store with real ACID transactions and strong (Raft-backed) consistency, not eventual consistency.
- You are already running TiDB, or want a proven storage layer you can grow to 100+ TB.
- You want geo-distributed replication with automatic sharding and rebalancing handled for you.
- You value an open governance model (CNCF graduated) over a single-vendor cloud store.

**Avoid when:**
- You want a simple embedded or single-node KV store — the PD + Raft + RocksDB operational cost is enormous overkill. Reach for RocksDB, Badger, or Redis instead.
- Your team can't own database-cluster operations (compaction tuning, capacity planning, upgrade orchestration).
- You need sub-millisecond point-read latency under contention — the TSO round-trip and LSM read path make this hard to guarantee.
- You only need a cache or session store; the transactional machinery buys you nothing there.

## Alternatives

- cockroachdb/cockroach — distributed SQL over a similar Raft+RocksDB(Pebble) design; use it when you want SQL and transactions in one system rather than a KV layer plus TiDB.
- etcd-io/etcd — Raft KV for configuration and coordination; use when you need a small, strongly consistent metadata store, not a multi-TB database.
- facebook/rocksdb — the embedded engine TiKV is built on; use directly when you want single-node storage without distribution.
- yugabyte/yugabyte-db — another Spanner-inspired distributed database; use when you want Postgres/Cassandra API compatibility built in.
- foundationdb/foundationdb — transactional distributed KV with a strong testing story; use when you want to build your own layers on a minimal ordered KV core.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-04 | First public release as TiDB's storage layer[^1]. |
| 1.0 | 2017-10 | First GA, shipped alongside TiDB 1.0. |
| 2.0 | 2018-04 | Region merge, performance and stability work. |
| 3.0 | 2019-06 | Pessimistic transactions, Titan (blob) storage engine. |
| — | 2020-09 | Graduated within the CNCF[^3]. |
| 5.0 | 2021-04 | Async commit, improved transaction latency. |
| 6.x | 2022 | Partitioned Raft KV engine work, scaling improvements. |
| 7.x–8.x | 2023–2025 | Ongoing releases tracking the TiDB version line. |

## References

[^1]: TiKV README and project history. https://github.com/tikv/tikv
[^2]: "Deep Dive TiKV" — architecture, Percolator transactions, Raft. https://tikv.org/deep-dive/introduction/
[^3]: CNCF, "Cloud Native Computing Foundation Announces TiKV Graduation" — 2020-09-02. https://www.cncf.io/announcements/2020/09/02/cloud-native-computing-foundation-announces-tikv-graduation/
[^4]: TiKV client drivers (Go / Java / Rust / C). https://tikv.org/docs/latest/reference/clients/introduction/

## Tags

rust, distributed-database, key-value-store, transactions, raft, rocksdb, cncf, tidb, consensus, storage-engine
