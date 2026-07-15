# apple/foundationdb

> An ordered, transactional key-value store that runs its entire cluster inside a deterministic simulator to prove correctness before shipping.

[GitHub repo](https://github.com/apple/foundationdb) ·
[Official website](https://apple.github.io/foundationdb/) ·
[License: Apache-2.0](https://github.com/apple/foundationdb/blob/main/LICENSE)

## Overview

FoundationDB (FDB) is a distributed database that exposes a single, flat, lexicographically ordered key-value keyspace with fully ACID, strictly serializable transactions across the entire cluster — not per-shard[^2]. It was built by FoundationDB Inc. (founded 2009), acquired by Apple in 2015 — at which point public downloads were discontinued — and open-sourced under Apache-2.0 in April 2018[^1]. Apple runs it under CloudKit; Snowflake uses it as its metadata store.

The defining tension is deliberate minimalism. The core provides only a key-value API: `get`, `getRange`, `set`, `clear`, atomic mutations, and `watch`. There is no query language, no secondary indexes, no schema, and no joins. Richer data models are meant to be built on top as stateless "layers" — the Record Layer (structured records with indexes, built by Apple for CloudKit) and the Document Layer (a MongoDB-compatible API) are the canonical examples[^5]. You get a very strong transactional substrate and are expected to build the database you actually want above it.

The second defining trait is how it is tested. FDB is written in Flow, a C++ dialect adding async/await and an actor model that transpiles to plain C++ via an `actorcompiler`[^2]. Because all concurrency and I/O go through Flow, the whole distributed system can be run single-threaded inside a deterministic discrete-event simulator with injected disk faults, network partitions, clock skew, and process kills ("Buggify"), replayable from a seed[^3]. This simulation-first methodology — not the feature set — is the project's real reputation, and the founders later spun it into the Antithesis testing company.

## Getting Started

FDB is a server you run, plus a client library your app links. On macOS/Linux, install the server and client packages from the [releases page](https://github.com/apple/foundationdb/releases), then use `fdbcli` to talk to the local cluster:

```bash
fdbcli
# fdb> status
# fdb> configure new single ssd   # single-machine dev config
```

Python client (every binding follows the same read-your-writes transaction model):

```python
import fdb
fdb.api_version(730)
db = fdb.open()

@fdb.transactional          # retries automatically on conflict / retryable errors
def set_and_read(tr):
    tr[b"class/intro"] = b"available"
    return tr[b"class/intro"]

print(set_and_read(db))     # b"available"
```

The `@fdb.transactional` decorator is central: a transaction that fails with a retryable error (conflict, `transaction_too_old`, etc.) is re-run from scratch, so the function body must be idempotent and side-effect-free apart from its reads and writes.

## Architecture / How It Works

A commit flows through several specialized roles rather than one monolithic node[^2]:

- **Coordinators** — a small Paxos group holding the cluster's configuration and electing the cluster controller. Losing a quorum of coordinators makes the cluster unavailable.
- **Cluster Controller** — recruits and monitors all other roles.
- **GRV Proxies / Commit Proxies** — clients ask a GRV proxy for a read version, then send buffered writes to a commit proxy at commit time.
- **Resolvers** — perform optimistic-concurrency conflict detection: if another transaction wrote a key you read within your read window, your commit is rejected.
- **Transaction Logs (tlogs)** — durably persist committed mutations (the write path's fsync point) before acknowledging.
- **Storage Servers** — asynchronously pull mutations from the tlogs and hold the actual data; reads are served here via MVCC.

Reads are lock-free and versioned: a transaction takes a global read version, and storage servers keep roughly the last five seconds of versions. Writes are buffered entirely on the client and validated only at commit — so FDB uses **optimistic concurrency control**, not locking. Data distribution automatically splits and rebalances the keyspace into shards across storage servers as data grows or hardware changes.

Storage engines are pluggable: `ssd` (a B-tree, historically SQLite-derived), `ssd-redwood` (a newer prefix-compressing B-tree), `memory` (RAM-resident, size-bounded), and a RocksDB-backed engine. Replication is configured as `single`/`double`/`triple`; a single cluster lives in one region, and multi-region deployments use asynchronous replication with satellite tlogs.

## Production Notes

The published limits are load-bearing and bite real workloads[^4]:

- **Five-second transaction limit.** A transaction older than ~5 s fails with `transaction_too_old` (error 1007) because storage servers only retain that version window. Long scans, big migrations, and analytical reads must be chunked across many transactions using range continuation.
- **10 MB transaction size cap.** A single transaction's writes cannot exceed 10 MB (with soft-throttling well below that). Bulk loading means batching thousands of small transactions, not one big one.
- **Key/value size caps.** Keys up to 10 KB, values up to 100 KB, but performance guidance is much smaller (keep values well under ~10 KB); large blobs belong split across keys or in an external store.
- **Hot keys and contention.** Because concurrency is optimistic, high-contention keys produce `not_committed` (1020) conflict errors and retry storms. Sequential keys (timestamps, auto-increment IDs) create write hotspots on one storage server; keyspace design (prefix sharding, randomized suffixes) is a first-class concern.
- **Ratekeeper throttling.** FDB deliberately throttles clients (raising GRV latency) to keep the cluster from falling behind on durability or data movement. Rising read-version latency is the primary "cluster under stress" signal in `status`.
- **Client/version coupling.** The client library version must be compatible with the cluster; the multi-version-client mechanism exists specifically to survive rolling upgrades. Upgrades are one major version at a time — the README's recommended path is `6.3.x → 7.1.x → 7.3.x`, and skipping a major release is only simulation-tested, not production-tested.

Operationally you get `fdbcli` (`status`, `configure`, `coordinators`, `exclude`/`include` for safe machine removal), `fdbbackup`/`fdbdr` for backup and cross-cluster DR, and `fdbmonitor` supervising `fdbserver` processes. Observability is thinner than mature SQL databases: much diagnosis happens through `status json` and trace-event logs rather than rich dashboards.

## When to Use / When Not

**Use when:**
- You need genuinely multi-key, cross-shard ACID with strict serializability and are willing to build your data model on top.
- Correctness under partition/hardware failure is paramount and you value the simulation-tested track record.
- You are building a stateful platform (metadata service, control plane, object index) and can encapsulate access behind a layer.

**Avoid when:**
- You want SQL, secondary indexes, or an ORM out of the box — FDB gives you none of these natively.
- Your access pattern needs large values, long-running analytical transactions, or streaming scans as first-class operations.
- Your team lacks the appetite to design keyspaces and a layer, and to operate a multi-role distributed system.

## Alternatives

- cockroachdb/cockroach — reach for this when you want distributed serializable transactions with SQL, indexes, and schema built in rather than assembled on a KV layer.
- tikv/tikv — a distributed transactional KV (Raft-based, backs TiDB) when you want a similar substrate with a ready-made SQL layer option.
- yugabyte/yugabyte-db — choose this for distributed SQL with PostgreSQL wire compatibility instead of a bespoke layer.
- etcd-io/etcd — use for strongly-consistent configuration and coordination at modest data sizes, not high-volume application data.
- scylladb/scylladb — pick this for very high-throughput wide-column workloads where eventual/tunable consistency is acceptable and cross-key ACID is not required.

## History

| Version | Date | Notes |
|---------|------|-------|
| Founded | 2009 | FoundationDB Inc. begins; deterministic-simulation approach from the start[^3]. |
| Acquired | 2015-03 | Apple acquires FoundationDB; public downloads discontinued. |
| Open-sourced | 2018-04 | Released on GitHub under Apache-2.0[^1]. |
| 6.0 | 2018 | Multi-region asynchronous replication. |
| 6.3 | ~2020 | Later marked unsupported; last patch 6.3.25. |
| 7.1 | ~2022 | Redwood storage engine maturation; long-lived stable branch (7.1.57). |
| 7.3 | current | Actively supported branch (7.3.69 latest production release). |

## References

[^1]: "FoundationDB is Open Source" — FoundationDB blog, 2018-04-19. https://www.foundationdb.org/blog/foundationdb-is-open-source/
[^2]: FoundationDB architecture documentation. https://apple.github.io/foundationdb/architecture.html
[^3]: Will Wilson, "Testing Distributed Systems w/ Deterministic Simulation" — Strange Loop 2014. https://www.youtube.com/watch?v=4fFDFbi3toc
[^4]: FoundationDB known limitations. https://apple.github.io/foundationdb/known-limitations.html
[^5]: FoundationDB Record Layer. https://github.com/FoundationDB/fdb-record-layer

## Tags

c-plus-plus, distributed-database, key-value-store, transactional, acid, nosql, storage-engine, deterministic-simulation, database, apple
