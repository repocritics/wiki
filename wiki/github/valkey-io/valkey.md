# valkey-io/valkey

> The Linux Foundation fork of Redis, kept under a permissive BSD license after Redis went source-available in 2024.

[GitHub repo](https://github.com/valkey-io/valkey) ·
[Official website](https://valkey.io) ·
[License: BSD-3-Clause](https://github.com/valkey-io/valkey/blob/unstable/COPYING)

## Overview

Valkey is an in-memory data structure server — the same category as Redis, and in fact a direct fork of it. It was created in March 2024 when Redis Inc. relicensed Redis from BSD to the dual SSPL / RSALv2 source-available licenses; the community fork was taken from the last BSD commit (Redis 7.2.4) and adopted by the Linux Foundation days later[^1]. Major cloud vendors that had been shipping managed Redis — AWS, Google Cloud, Oracle, Ericsson, Snap — moved their engineering behind Valkey rather than a source-available Redis[^2].

The defining fact about Valkey is therefore governance, not technology. It is protocol-compatible with Redis (RESP2/RESP3), reuses the same data model and most of the same source, and `make install` even creates `redis-server` / `redis-cli` symlinks for drop-in replacement. What differs is the license and the neutral, multi-vendor stewardship under LF Projects. For teams whose objection to Redis was legal rather than functional, Valkey is the low-friction path.

Since the fork the two codebases have diverged. Valkey 8.0 (2024) added async I/O threading, dual-channel replication and per-slot dictionaries, claiming large throughput gains over the 7.2 baseline[^3]. Redis, for its part, relicensed its core back to open source (AGPLv3, added as an option) with Redis 8 in 2025 — so the "only Valkey is open" framing that drove the 2024 migration is now more nuanced, and the projects compete on merit as well as license.

## Getting Started

Valkey is C and builds from source with a plain `Makefile`; there is no dependency manager to install first. On Debian/Ubuntu you need `build-essential` (and `libssl-dev` for TLS).

```bash
git clone https://github.com/valkey-io/valkey.git
cd valkey
make                    # add BUILD_TLS=yes for TLS
make install            # installs valkey-server, valkey-cli, redis-* symlinks
```

```bash
# start a server, then in another shell:
valkey-server --port 6379 &
valkey-cli
```

```
127.0.0.1:6379> SET user:1 "Tom"
OK
127.0.0.1:6379> INCR page:views
(integer) 1
127.0.0.1:6379> EXPIRE user:1 3600
(integer) 1
```

Most Redis client libraries connect unchanged — point them at the same host/port. Managed options exist as AWS ElastiCache / MemoryDB for Valkey, Google Cloud Memorystore for Valkey, and Aiven.

## Architecture / How It Works

The core is inherited from Redis and worth understanding on its own terms:

- **Single-threaded command execution.** Commands run on one main event-loop thread, which is what makes each command atomic and keeps semantics simple. This is unchanged in Valkey — the multi-threading added in 8.0 is *I/O* threading (socket read/write and now parts of command parsing), not parallel execution of the command logic itself. A single expensive command still blocks every other client.
- **Data structures live in RAM.** Strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, and geospatial indexes are all in-process memory structures. Disk is used only for durability, not as the working set.
- **Persistence is optional and dual-form.** RDB point-in-time snapshots (via `fork()` + copy-on-write) and/or AOF (append-only command log, replayed on restart). You can run with neither for a pure cache.
- **Replication is asynchronous.** A primary streams writes to replicas; failover promotes a replica. Valkey 8.0 added *dual-channel replication* to separate the RDB transfer from the live command stream during full syncs, reducing primary memory/CPU pressure during replica bootstrap[^3].
- **Cluster mode** shards the keyspace across 16384 hash slots, with nodes discovering each other via a gossip protocol. Valkey 8.0 replaced the single cluster dictionary with per-slot dictionaries, which speeds up slot migration and defragmentation.
- **Extensibility** comes from the module API (loadable `.so`s), a Lua scripting engine, and functions. The project maintains companion modules — valkey-json, valkey-search (vector/secondary index), valkey-bloom — as separate repos rather than a bundled "Stack".

The allocator matters: jemalloc is the default on Linux (chosen for lower fragmentation than libc malloc), configurable via `MALLOC=`. TLS and experimental RDMA transport can be compiled either built-in or as modules.

## Production Notes

**It is a drop-in Redis replacement, up to a point.** Because Valkey forked at Redis 7.2.4, behavior is identical through that version; anything relying on Redis 7.4+ or Redis 8 features (or the commercial Stack modules) is not automatically covered. Validate your client library, your `KEYS`/`SCAN` patterns, and any Lua scripts against the specific Valkey version, not "Redis in general."

**The single-threaded core is the classic footgun.** `KEYS *`, unbounded `SORT`, large `HGETALL`, `SMEMBERS` on huge sets, and slow Lua scripts block the entire server for their duration. Use `SCAN`/`HSCAN` cursors, cap collection sizes, and watch the slowlog. I/O threads help throughput on many-connection workloads but do not rescue you from one heavy command.

**Memory is the resource to plan around.** Set `maxmemory` and an eviction policy explicitly; the default (`noeviction`) turns a full instance into write errors. Fragmentation is real — monitor `mem_fragmentation_ratio`. The `fork()` used for RDB snapshots and AOF rewrites can transiently near-double RSS under heavy writes (copy-on-write page churn) and cause latency spikes; size hosts for the fork, not just the dataset.

**Durability is best-effort by default.** Async replication means a window of un-replicated writes is lost on failover; `WAIT` and `appendfsync always` tighten this at a throughput cost. There is no synchronous cross-node consensus — Valkey is an AP-leaning cache/store, not a strongly-consistent database.

**Cluster constraints.** Multi-key operations (`MGET`, transactions, Lua touching multiple keys) must land in the same slot; use hash tags `{...}` to co-locate related keys. Cross-slot transactions are rejected. Resharding is online but adds load.

**Upgrading from Redis.** Migrating off Redis ≤ 7.2.4 is generally a binary swap — RDB and AOF files are compatible. The `redis-server`/`redis-cli` symlinks mean init scripts and tooling often need no change. Migrating off Redis 7.4+ or Redis 8 requires checking for features that never existed in the fork.

**Default branch is `unstable`.** Like upstream Redis, development happens on `unstable`; do not build production from it. Pin to a released tag.

## When to Use / When Not

**Use when:**
- You want Redis semantics and performance but need a permissively-licensed (BSD-3), vendor-neutral project.
- You already run Redis 7.2-era workloads and want a low-risk migration with the same clients and data files.
- You want first-class support on AWS/GCP managed offerings that standardized on Valkey.
- You need caching, session storage, rate limiting, leaderboards, queues, pub/sub, or streams in memory.

**Avoid when:**
- Your working set is much larger than RAM and you can tolerate disk latency — a disk-backed store fits better.
- You need strong consistency / durable transactions — this is an async-replicated in-memory store, not a system of record.
- You depend on Redis Inc.'s commercial Stack modules or Redis 8-specific features; the module ecosystems differ.
- You want vertical scaling across many cores from a single node without cluster mode — the command core is single-threaded by design.

## Alternatives

- redis/redis — the upstream Valkey forked from; relicensed to add AGPLv3 in 2025. Use it if you want the original brand, the commercial Stack modules, or Redis 8+ features and the AGPL terms are acceptable.
- dragonflydb/dragonfly — multi-threaded, Redis-compatible, scales vertically on one large node; use it when a single beefy instance beats a cluster, and the BSL license is acceptable.
- microsoft/garnet — C#, Redis-protocol-compatible, tiered storage and high multi-core throughput; use when you want disk tiering and .NET operability.
- apache/kvrocks — RocksDB-backed, disk-first, speaks the Redis protocol; use when the dataset must exceed RAM cheaply.
- memcached/memcached — pure multi-threaded cache, no persistence or rich data types; use when you only need a fast key/value cache and nothing else.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2024-03 | Fork announced from Redis 7.2.4 after Redis relicensing; joins Linux Foundation[^1]. |
| 7.2.5 | 2024-04 | First Valkey release; drop-in compatible with Redis 7.2.4[^2]. |
| 8.0 | 2024-09 | Async I/O threading, dual-channel replication, per-slot dictionaries, experimental RDMA[^3]. |
| 8.1 | 2025 | Further memory-efficiency and replication work; companion modules (valkey-search, valkey-bloom) matured. |

## References

[^1]: Linux Foundation, "Open Source In-Memory Storage: Valkey" — 2024-03-28. https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community
[^2]: Valkey project, "Valkey" — history and licensing. https://valkey.io/
[^3]: Valkey blog, "Valkey 8.0" release overview. https://valkey.io/blog/

## Tags

c, in-memory-database, key-value-store, cache, redis-fork, nosql, distributed-systems, database, linux-foundation, pub-sub
