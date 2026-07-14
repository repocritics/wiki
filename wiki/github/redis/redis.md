# redis/redis

> In-memory data structure server — key-value store, cache, message broker, and, increasingly, a document and vector query engine.

[GitHub repo](https://github.com/redis/redis) ·
[Official website](https://redis.io) ·
[License: RSALv2 OR SSPLv1 OR AGPLv3](https://github.com/redis/redis/blob/unstable/LICENSE.txt)

## Overview

Redis (REmote DIctionary Server) is a single-process, in-memory data store written in C, created by Salvatore Sanfilippo (antirez) in 2009[^1]. Its defining idea is that the server understands its values: instead of opaque blobs, it exposes native data structures — strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLogs — with atomic server-side operations on each. That is why it is described as a "data structure server" rather than a cache: a leaderboard is a sorted set with `ZADD`/`ZRANGE`, a rate limiter is an `INCR` with `EXPIRE`, a work queue is a list with `BLPOP`.

The central tradeoff is memory versus everything else. Redis keeps the working set in RAM to get sub-millisecond latency, which makes it fast and simple but caps dataset size at available memory and makes durability a configuration decision rather than a default. It occupies an unusual position: fast enough and structured enough to be a primary datastore for some workloads, but most often deployed as a cache or coordination layer in front of a disk-backed database of record.

Since 2024 Redis has also been the center of an open-source licensing dispute. Redis 7.4 (2024) moved from the BSD-3-Clause license to a source-available dual license (RSALv2 / SSPLv1), which is not OSI-approved; this triggered the [valkey-io/valkey](https://github.com/valkey-io/valkey) fork under the Linux Foundation, backed by AWS, Google, and Oracle[^2]. Redis 8.0 (2025) added the AGPLv3 as a third option — an OSI-approved license — folded the previously separate Stack modules (Search, JSON, Time Series, Bloom) into the core distribution, and renamed "Redis Community Edition" to "Redis Open Source"[^3]. antirez, who had stepped down as maintainer in 2020, rejoined Redis Ltd. in 2024 and shipped vector sets as a new type.

## Getting Started

```bash
# Docker (fastest path)
docker run -d -p 6379:6379 redis:latest

# macOS
brew install redis && redis-server
```

```text
$ redis-cli
redis> SET user:1 "Tom"
OK
redis> INCR page:views          # atomic counter
(integer) 1
redis> ZADD leaderboard 100 tom 90 brad   # sorted set
(integer) 2
redis> ZREVRANGE leaderboard 0 -1 WITHSCORES
1) "tom"
2) "100"
3) "brad"
4) "90"
redis> EXPIRE user:1 3600       # TTL in seconds
(integer) 1
```

Applications connect through a client library — `redis-py` (Python), `node-redis` / `ioredis` (Node), `go-redis` (Go), `Jedis` / `Lettuce` (Java), `StackExchange.Redis` (.NET) — all speaking the same RESP wire protocol over a TCP socket.

## Architecture / How It Works

**Single-threaded command execution.** The core is an event loop (`ae`) that processes commands one at a time. This is deliberate: with no locks and no context switching, each command is atomic by construction, and the data structures need no internal synchronization. Throughput comes from keeping every operation O(1) or O(log n) and never blocking the loop. The corollary is that a single slow command (`KEYS *`, a large `SORT`, a Lua script in a loop) stalls every other client. Redis 6.0 added threaded I/O to parallelize socket reads/writes and reply encoding, but command execution itself remains on the main thread[^4].

**Encodings.** Each logical type has multiple internal representations chosen by size. Small hashes, lists, and sorted sets use compact `listpack`/`ziplist` layouts; they convert to hash tables, `quicklist`, or skiplists once they cross configurable thresholds (`hash-max-listpack-entries`, etc.). Small integer sets use `intset`. This memory-vs-CPU tradeoff is transparent but shows up in `OBJECT ENCODING` and in latency cliffs when a structure crosses a threshold.

**Persistence.** Two mechanisms, often combined: RDB point-in-time snapshots (`fork()` + copy-on-write, compact, fast to load) and AOF (append-only file logging every write, replayed on restart). AOF is more durable but larger and slower to load; `appendfsync everysec` is the common middle ground. Neither makes Redis a fully ACID-durable store — a crash between fsyncs loses recent writes.

**Replication and HA.** Asynchronous primary-replica replication (`PSYNC`) with a replication backlog for partial resync. High availability for non-sharded setups uses Redis Sentinel (a separate process that monitors and fails over). Horizontal scaling uses Redis Cluster: keys are hashed into 16384 slots distributed across primaries, clients are redirected via `MOVED`/`ASK`, and multi-key operations must land in the same slot (enforced with hash tags like `{user1}`).

**Programmability.** Lua scripting (`EVAL`) runs atomically on the server; Redis 7.0 added Functions (`FUNCTION LOAD`) for persisted, named server-side logic. The Modules API (C) is how RediSearch, RedisJSON, and the probabilistic types extend the command set — as of Redis 8 these ship inside the core distribution rather than as add-ons.

## Production Notes

**Memory is the hard limit and the main footgun.** Set `maxmemory` and an eviction policy (`allkeys-lru`, `volatile-ttl`, etc.) explicitly — the default (`noeviction`) makes writes fail with OOM once memory fills. Fragmentation (jemalloc) can make RSS exceed logical dataset size by 1.3–1.5×; watch `mem_fragmentation_ratio`. `fork()` for RDB/AOF-rewrite transiently doubles memory under heavy write load because of copy-on-write page churn.

**Latency spikes.** The usual culprits: O(n) commands on large keys (`KEYS`, `SMEMBERS`, `HGETALL`, `DEL` on a huge collection — use `UNLINK` for async free), synchronous `SAVE`, AOF rewrites, and `fork()` pauses on large datasets. `SCAN`/`HSCAN`/`SSCAN` exist specifically to avoid blocking iteration. Enable the slow log and `LATENCY DOSCTOR`.

**Persistence is a choice, not a default guarantee.** A default `redis.conf` gives you RDB snapshots on an interval — meaning a crash can lose minutes of writes. If Redis is your system of record, enable AOF and understand the fsync tradeoff. Many teams instead treat Redis as disposable cache and accept cold-start rebuild.

**Redis Cluster constraints.** No cross-slot transactions or multi-key commands unless keys share a hash tag; Lua scripts must declare keys and stay within one slot. Client libraries vary in cluster-topology handling. Sentinel and Cluster solve different problems and are not interchangeable.

**Licensing due diligence.** Versions 7.4 through 8.0-pre are source-available only (RSALv2/SSPLv1) — relevant if you build a managed Redis offering or need OSI-approved licensing for compliance. Redis 8.0+ restores an OSI option via AGPLv3, but AGPL's network-copyleft terms are themselves a review item for some organizations. Redis 7.2 and earlier remain BSD-3-Clause. [valkey-io/valkey](https://github.com/valkey-io/valkey) is the BSD-licensed fork of the 7.2 codebase for teams that want to stay on a permissive license.

## When to Use / When Not

**Use when:**
- You need sub-millisecond reads/writes for a working set that fits in RAM (caching, sessions, rate limiting, leaderboards).
- You want atomic server-side data-structure operations instead of read-modify-write round trips.
- You need lightweight pub/sub, streams, or a simple job queue without standing up Kafka.
- You need a shared coordination primitive (locks, counters, feature flags) across app instances.

**Avoid when:**
- Your dataset is much larger than affordable RAM and is not naturally partitionable — a disk-first database is cheaper per byte.
- You need strong durability / ACID transactions as a hard requirement (Redis transactions are `MULTI`/`EXEC` batching, not rollback-capable ACID).
- You need rich ad-hoc querying and joins — even with Redis Search, this is not a relational engine.
- Your organization cannot accept RSALv2/SSPLv1/AGPLv3 terms and you cannot pin to BSD-era 7.2 or Valkey.

## Alternatives

- [valkey-io/valkey](https://github.com/valkey-io/valkey) — BSD-licensed fork of Redis 7.2 under the Linux Foundation; use when you want the Redis API and permissive licensing.
- [memcached/memcached](https://github.com/memcached/memcached) — simpler multi-threaded cache; use when you only need a plain key-value LRU cache and want horizontal thread scaling.
- [apache/kafka](https://github.com/apache/kafka) — durable, partitioned log; use instead of Redis Streams/pub-sub when you need retention, replay, and high-throughput event streaming.
- [dragonflydb/dragonfly](https://github.com/dragonflydb/dragonfly) — Redis-API-compatible, multi-threaded, vertically scalable server; use when a single node needs more cores than Redis's single execution thread can use.
- [etcd-io/etcd](https://github.com/etcd-io/etcd) — strongly consistent (Raft) key-value store; use for configuration/coordination where correctness beats latency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009 | Initial release by antirez[^1]. |
| 2.6 | 2012 | Lua scripting (`EVAL`). |
| 3.0 | 2015-04 | Redis Cluster (16384 hash slots). |
| 4.0 | 2017-07 | Modules API, PSYNC2, LFU eviction, `UNLINK`. |
| 5.0 | 2018-10 | Streams data type. |
| 6.0 | 2020-04 | ACLs, RESP3, threaded I/O, TLS. |
| 7.0 | 2022-04 | Functions, sharded pub/sub, ACL v2. |
| 7.2 | 2023-08 | Last BSD-3-Clause release. |
| 7.4 | 2024 | License change to RSALv2/SSPLv1; hash field TTL. Valkey fork follows[^2]. |
| 8.0 | 2025 | AGPLv3 added; Stack modules folded into core; vector sets (beta); renamed Redis Open Source[^3]. |

## References

[^1]: Salvatore Sanfilippo, Redis project history and origin. https://redis.io/about/
[^2]: Redis license change and the Valkey fork under the Linux Foundation, 2024. https://valkey.io/blog/introducing-valkey/
[^3]: Redis, "Redis is open source again" / Redis 8 licensing and Redis Open Source rename, 2025. https://redis.io/blog/agplv3/
[^4]: Redis documentation, "Redis is single threaded — how can I exploit multiple CPUs?" and threaded I/O notes. https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/

## Tags

in-memory-database, key-value-store, cache, c, data-structure-server, message-broker, pub-sub, vector-database, nosql, distributed-systems, database
