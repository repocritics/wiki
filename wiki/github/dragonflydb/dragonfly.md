# dragonflydb/dragonfly

> A multi-threaded, Redis- and Memcached-compatible in-memory data store that scales vertically on a single node instead of horizontally across a cluster.

[GitHub repo](https://github.com/dragonflydb/dragonfly) ·
[Official website](https://www.dragonflydb.io/) ·
[License: BSL 1.1](https://github.com/dragonflydb/dragonfly/blob/main/LICENSE.md)

## Overview

Dragonfly is an in-memory key-value store that speaks the Redis (RESP) and Memcached wire protocols, so existing clients connect with no code changes[^1]. Its thesis is architectural: where Redis runs one command loop per process and scales by sharding across many processes/nodes (Redis Cluster), Dragonfly is multi-threaded and partitions the keyspace across cores inside a single process. The pitch is that one large Dragonfly instance can replace a fleet of Redis shards, with the project claiming up to 25× the throughput of single-process Redis on large instances[^2]. The vendor's own benchmarks are the source of that figure; treat them as best-case, network-saturated numbers, not a guarantee for your workload.

The project was started in 2022 by Roman Gershman (romange), previously an engineer at Redis Labs and Google, as an experiment in what an in-memory store designed around modern cloud hardware (many cores, io_uring, large RAM) would look like[^3]. It is backed by a commercial company, DragonflyDB, Ltd., which sells Dragonfly Cloud.

The defining tension is licensing and scaling model. Dragonfly is **not OSI open source**: it ships under the Business Source License 1.1, which forbids offering it as a competing managed service and converts to Apache 2.0 only on 2030-07-01[^4]. And the single-node vertical-scaling model is a genuine tradeoff — it removes cluster operational complexity but concentrates your data on one machine's limits.

## Getting Started

The supported path is the official Docker image:

```bash
docker run --network=host --ulimit memlock=-1 \
  docker.dragonflydb.io/dragonflydb/dragonfly
```

Then connect with any Redis client:

```bash
redis-cli -p 6379
> SET user:1 "Tom"
OK
> GET user:1
"Tom"
> INFO server        # returns Dragonfly-shaped fields
```

Common production flags (native binary):

```bash
./dragonfly --logtostderr --requirepass=changeme \
  --cache_mode=true --maxmemory=12gb \
  --dbfilename dump --snapshot_cron="0 * * * *"
```

`--cache_mode=true` turns on the adaptive eviction algorithm; without it Dragonfly returns OOM errors at `maxmemory` rather than evicting[^1].

## Architecture / How It Works

Dragonfly is built on **shared-nothing** principles borrowed from database design[^3]. The keyspace is split into shards, and each shard is owned exclusively by one thread. A key always maps to the same shard, so single-key operations never contend on locks — the owning thread processes them serially and lock-free.

- **Threading / I/O** — Dragonfly runs on `helio`[^5], the author's own fiber + I/O framework, which uses `io_uring` on modern Linux and falls back to `epoll` elsewhere. Concurrency inside a thread is expressed with user-space fibers, not OS threads.
- **Transactions** — multi-key and multi-command atomicity is provided by a transactional framework derived from the VLL lock-manager paper[^6]. A transaction that spans several shards acquires per-shard locks in a deadlock-free order and hops between the relevant threads, giving atomic `MULTI`/`EXEC`, `MSET`, and Lua execution without global mutexes.
- **Hash table** — the core dictionary is `Dashtable`, an adaptation of the "Dash" extendible-hashing design[^7]. It preserves the incremental-rehash and stateless-`SCAN` semantics of Redis while using less memory and CPU per entry.
- **Snapshotting** — Dragonfly uses a **fork-less**, point-in-time snapshot algorithm. Unlike Redis's `BGSAVE`, which `fork()`s and can transiently double memory under write load via copy-on-write, Dragonfly snapshots in-process with near-flat memory. It can emit RDB-compatible dumps and its own multi-file `dfs` format.

The cost of the per-shard model: a single hot key lives on exactly one thread, so a workload dominated by one key cannot use more than one core (the classic hot-key problem). Cross-shard transactions and global Lua scripts also serialize across the shards they touch, eroding the multi-threaded advantage the more keys they span.

## Production Notes

**Licensing is the first thing to check.** BSL 1.1 is source-available, not open source. The Additional Use Grant lets you run Dragonfly for your own product or service but prohibits offering it *as* a managed database service to third parties[^4]. If your legal/procurement process requires an OSI-approved license, Dragonfly does not qualify until its 2030 Apache-2.0 change date. This is the same class of restriction that drove the Redis/Valkey split.

**Durability is snapshot-based.** Dragonfly's persistence story centers on periodic snapshots (`snapshot_cron`) plus optional replication. It has not historically offered a Redis-style append-only file (AOF) with per-second fsync, so the worst-case data-loss window is "time since last snapshot," not "one second." Verify current AOF/journaling status against the docs for your version before relying on it for durability[^1].

**Kernel and container setup matter.** For the io_uring fast path you want a reasonably modern Linux kernel (5.x+). Containers must raise the memlock ulimit (`--ulimit memlock=-1`) or startup fails. On macOS and older kernels Dragonfly runs but on the epoll fallback, so local performance is not representative of Linux production.

**No Redis modules.** Dragonfly does not load Redis `.so` modules (RedisJSON, RediSearch, RedisBloom, etc.). It reimplements some of that surface natively (JSON, and a built-in vector/search capability), but if your app `MODULE LOAD`s third-party modules, that will not work — you must map each feature to Dragonfly's native equivalent or drop it.

**Vertical scaling is the happy path.** The intended deployment is one large node (many cores, lots of RAM) with a replica for failover. Multi-node clustering exists (`cluster_mode`, including an emulated single-node cluster and real multi-node modes in newer releases), but it is younger and less battle-tested than Redis Cluster's decade of production use. If you need geo-sharding across dozens of nodes, evaluate carefully.

**Compatibility is high but not total.** Command coverage is broad and grows each release, but it is a reimplementation — behavioral edge cases (exact error strings, `INFO` fields, some `OBJECT ENCODING` values, keyspace-notification details) can differ from a given Redis version. Millisecond expirations above ~2^28 ms are rounded to the nearest second (<0.001% error)[^1]. Test compatibility-sensitive code paths rather than assuming byte-for-byte parity.

## When to Use / When Not

**Use when:**
- You're outgrowing a single Redis process and want to scale up on one big machine instead of standing up and operating Redis Cluster.
- Your snapshot/`BGSAVE` memory spikes or fork stalls are causing latency or OOMs — Dragonfly's fork-less snapshotting directly targets this.
- You want higher throughput per node with the same Redis/Memcached clients and no application rewrite.
- You're memory-constrained and want a more compact keyspace footprint.

**Avoid when:**
- You need an OSI-approved open-source license, or you intend to offer the store as a managed service (BSL forbids it until 2030).
- You depend on AOF-grade durability, third-party Redis modules, or exact Redis behavioral parity.
- Your workload is dominated by one or a few hot keys — the per-shard threading model can't parallelize that.
- You already run a large, mature Redis Cluster and horizontal geo-sharding is a hard requirement.

## Alternatives

- redis/redis — the incumbent; single-threaded per process, unmatched ecosystem and module support, now itself under a source-available license (RSALv2/SSPL for recent versions).
- valkey-io/valkey — Linux Foundation BSD-licensed fork of Redis 7.2; use when you specifically need a truly open-source Redis drop-in.
- microsoft/garnet — Microsoft's C#/.NET multi-threaded Redis-compatible store; another vertical-scaling option, MIT-licensed.
- Snapchat/KeyDB — multi-threaded fork of Redis (now under Snap); similar "faster Redis" pitch, less active development.
- apache/kvrocks — RocksDB-backed, disk-based Redis-protocol store; use when your dataset must exceed RAM and Redis-API compatibility matters more than in-memory latency.
- memcached/memcached — pure LRU cache; use when you only need caching and want the smallest, simplest server.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-12 | Repository created[^8]. |
| public launch | 2022-05 | Open-sourced; shared-nothing multi-threaded design announced[^3]. |
| 1.0 | 2023-03 | First production-ready GA release[^9]. |
| 1.39.0 | 2026-06-09 | Latest tagged release at time of writing[^8]. |

## References

[^1]: Dragonfly README and configuration reference. https://github.com/dragonflydb/dragonfly/blob/main/README.md
[^2]: Dragonfly benchmarks (vendor-published, up to 25× Redis / 3.8M QPS on c6gn.16xlarge). https://www.dragonflydb.io/
[^3]: "Background" section, Dragonfly README — shared-nothing architecture and 2022 design goals. https://github.com/dragonflydb/dragonfly/blob/main/README.md#background
[^4]: Dragonfly Business Source License 1.1 — Change Date 2030-07-01, Change License Apache 2.0. https://github.com/dragonflydb/dragonfly/blob/main/LICENSE.md
[^5]: helio — fiber and I/O framework powering Dragonfly. https://github.com/romange/helio
[^6]: "VLL: a lock manager redesign for main memory database systems." https://www.cs.umd.edu/~abadi/papers/vldbj-vll.pdf
[^7]: "Dash: Scalable Hashing on Persistent Memory." https://arxiv.org/pdf/2003.07302.pdf
[^8]: GitHub API, repos/dragonflydb/dragonfly and releases (fetched 2026-07-15).
[^9]: DragonflyDB — Dragonfly 1.0 general-availability announcement (2023-03). https://www.dragonflydb.io/blog

## Tags

cpp, in-memory-database, key-value-store, redis-compatible, memcached-compatible, cache, multi-threading, shared-nothing, nosql, vector-search, bsl-license, data-store
