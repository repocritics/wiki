# tarantool/tarantool

> An in-memory database and a Lua application server fused into one process — data lives in RAM and your stored procedures run next to it.

[GitHub repo](https://github.com/tarantool/tarantool) ·
[Official website](https://www.tarantool.io) ·
[License: BSD-2-Clause](https://github.com/tarantool/tarantool/blob/master/LICENSE)

## Overview

Tarantool is two things shipped as one binary: an in-memory transactional database and an application server built on a heavily patched LuaJIT 2.1[^1]. The pitch — "get your data in RAM, get compute close to data" — means you write server-side business logic in Lua that executes in the same process as the storage engine, avoiding the network hop between app tier and database that dominates latency in conventional stacks. Its C core has been in continuous development since 2010, originally at Mail.ru (now VK), where it runs large-scale production workloads[^2].

The defining tension is that Tarantool is neither a drop-in cache like Redis nor a general RDBMS like PostgreSQL — it occupies an unusual middle where you get MessagePack tuples, indexed spaces, ACID transactions, WAL persistence, replication, and (since 2.x) ANSI SQL, but you are also expected to embed application code in the datastore. Teams that adopt it for the latency win inherit its operational model: a single-threaded transaction core, RAM-bound working sets on the default engine, and a Lua-first ecosystem. Used well it collapses a whole tier; used as "just a faster Redis" it exposes footguns that a plain cache never would.

## Getting Started

```bash
# Docker (fastest path to a running instance)
docker run --rm -t -i -p 3301:3301 tarantool/tarantool

# Or install the binary package (Debian/Ubuntu example)
curl -L https://tarantool.io/release/3/installer.sh | bash
apt install tarantool
```

```lua
-- init.lua — configure storage, create a space, insert and query
box.cfg{ listen = 3301, memtx_memory = 256 * 1024 * 1024 }

-- A "space" is a table; create it once (idempotent guard shown)
box.schema.space.create('users', { if_not_exists = true })
box.space.users:format({
    { name = 'id',    type = 'unsigned' },
    { name = 'email', type = 'string'   },
})
box.space.users:create_index('primary', {
    parts = { 'id' }, if_not_exists = true,
})

box.space.users:insert{ 1, 'ada@example.com' }
local row = box.space.users:get(1)          -- indexed point lookup, in-RAM
print(row.email)                            -- ada@example.com
```

Stored procedures are ordinary Lua functions registered on `box.func` or exposed over the iproto binary protocol; a client in any supported connector calls them by name.

## Architecture / How It Works

Tarantool's core is a single **transaction (TX) thread** that owns all in-memory data and runs every transaction and every Lua stored procedure. Concurrency inside that thread is **cooperative multitasking via fibers**: lightweight coroutines that yield on I/O, giving non-blocking behavior without OS threads or locks. Separate OS threads handle WAL writing, network (iproto), and replication relays, but the data-mutating logic is serialized on one core. This is the single most important fact about operating Tarantool.

There are two storage engines, chosen per space:

- **memtx** — the default, 100% in-memory. All tuples live in a slab allocator; durability comes from an append-only **write-ahead log (WAL)** plus periodic **snapshots** (checkpoints). Recovery replays the latest snapshot then the WAL tail. Indexes are HASH, TREE, RTREE (spatial), and BITSET. Working set must fit in RAM. memtx gained a full MVCC transaction manager so interactive transactions can run without blocking readers[^3].
- **vinyl** — a disk-based LSM-tree engine for datasets larger than memory, with the usual LSM tradeoffs (write amplification, compaction, range-scan cost) in exchange for not being RAM-bound.

Data is stored and transmitted as **MessagePack**; the client-server protocol (iproto) is MessagePack-framed, which keeps serialization cheap on both ends. Replication is **row-based WAL shipping**: asynchronous master-master (or master-replica) by default, with **synchronous quorum-based replication** and **RAFT-based automatic leader election** available for single-leader configurations that need stronger consistency[^4]. Sharding is *not* in the core — it is provided by the separate `vshard` module, and cluster orchestration historically by `cartridge`.

Tarantool 3.0 (2024) introduced a **declarative YAML configuration** system that supersedes much of the imperative `box.cfg` + cartridge approach for cluster topology, replicasets, and roles[^5]. Older deployments still run the imperative model, so the ecosystem currently spans two configuration paradigms.

## Production Notes

**Single-threaded core is the master constraint.** One CPU-heavy Lua stored procedure that does not yield stalls *every* client on that instance. To use more than one core per host you run multiple Tarantool instances and shard across them with vshard — vertical scaling on a box means "more instances," not "more threads." Profile stored procedures for hidden non-yielding loops.

**memtx is RAM-bound and the arena is a hard wall.** `memtx_memory` caps the slab arena; hitting it returns `ER_MEMORY_ISSUE` and can wedge writes. Size for peak plus headroom, monitor `box.slab.info()`, and remember snapshots transiently increase memory pressure. Datasets that outgrow RAM belong on vinyl or must be sharded.

**Checkpointing and WAL are I/O events, not free.** Snapshots write the whole memtx dataset to disk; on large instances this is a periodic I/O and CPU spike. WAL fsync policy (`wal_mode`) trades durability for throughput — `none` is fast and lossy, `write` is the durable default. A slow disk turns the WAL thread into the bottleneck.

**Async master-master has no conflict resolution.** Multi-master replication ships rows optimistically; concurrent conflicting writes on two masters diverge, and Tarantool will not merge them for you. If you need correctness under concurrent writes, use synchronous replication with RAFT (a single leader) rather than active-active.

**Upgrades touch the on-disk schema.** Moving between major series generally requires `box.schema.upgrade()` after binaries are updated, and the snapshot/WAL format is version-tied — test recovery on a replica before upgrading the leader. The 1.10 → 2.x → 2.11 → 3.0 path each carried non-trivial migration steps, and 3.0's declarative config is a genuine operational shift, not a syntax swap.

**The ecosystem is Lua-first and Russian-origin.** Much production tooling (queue, expirationd, metrics) is Lua modules; connectors exist for major languages but the "native" experience is Lua. Documentation quality is good but some advanced material and community discussion originates in Russian.

## When to Use / When Not

**Use when:**
- Latency is dominated by the app↔database network hop and you can push logic into stored procedures.
- You want an in-memory store with real ACID transactions, WAL durability, and replication — not just a volatile cache.
- Your hot working set fits in RAM (memtx) or you accept LSM tradeoffs for larger sets (vinyl).
- You need queues, caching, and stateful web-backend state in one system and are comfortable operating Lua.

**Avoid when:**
- You want a simple, language-agnostic cache with no embedded application layer — Redis is less to operate.
- Your workload is CPU-bound per key/transaction and you cannot shard, since one core serializes the core.
- You need a mature, broadly-staffed SQL RDBMS with a deep operator talent pool — Postgres/MySQL hiring is easier.
- You are unwilling to run multiple instances to scale across cores, or to manage RAM sizing as a first-class concern.

## Alternatives

- redis/redis — use Redis when you want a simple in-memory cache/store with a huge client ecosystem and no embedded app server or SQL.
- apache/ignite — use Ignite when you want an in-memory computing platform on the JVM with distributed compute and SQL, and you are already JVM-centric.
- aerospike/aerospike-server — use Aerospike when you need a purpose-built low-latency KV store with hybrid RAM/SSD tiering and predictable operations at scale.
- scylladb/scylladb — use ScyllaDB when you need a distributed wide-column (Cassandra-model) store optimized for throughput, not an app+db fusion.
- dragonflydb/dragonfly — use Dragonfly when a single-threaded cache is your bottleneck and you want a multi-threaded, Redis-compatible drop-in.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010 | Project started at Mail.ru; in-memory store + Lua app server[^2]. |
| 1.6 | 2015 | Early stable series; MessagePack protocol, memtx maturity. |
| 1.10 | 2018 | LTS series; vinyl LSM engine, WAL-based replication hardened. |
| 2.0 | 2018 | SQL support added (beta), building on the box engine[^6]. |
| 2.11 | 2023 | LTS series; synchronous replication + RAFT leader election matured[^4]. |
| 3.0 | 2024 | Declarative YAML cluster configuration, superseding imperative/cartridge setup[^5]. |

## References

[^1]: Tarantool README — feature list, LuaJIT 2.1 basis, BSD-2-Clause. https://github.com/tarantool/tarantool
[^2]: Tarantool documentation, "About Tarantool" — origins and in-memory computing platform positioning. https://www.tarantool.io/en/doc/latest/
[^3]: Tarantool documentation, "Transaction model" — memtx MVCC transaction manager. https://www.tarantool.io/en/doc/latest/concepts/atomic/
[^4]: Tarantool documentation, "Replication" — asynchronous, synchronous quorum, and RAFT leader election. https://www.tarantool.io/en/doc/latest/concepts/replication/
[^5]: Tarantool documentation, "Configuration" — declarative YAML configuration introduced in 3.0. https://www.tarantool.io/en/doc/latest/reference/configuration/
[^6]: Tarantool documentation, "SQL" — ANSI SQL layer over the box engine. https://www.tarantool.io/en/doc/latest/reference/reference_sql/

## Tags

lua, c, in-memory-database, application-server, luajit, msgpack, lsm-tree, replication, nosql, sql, key-value-store, database
