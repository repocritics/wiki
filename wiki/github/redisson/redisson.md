# redisson/redisson

> A Java client for Valkey and Redis that exposes the server as distributed `java.util`-style objects and services rather than raw commands.

[GitHub repo](https://github.com/redisson/redisson) ·
[Official website](https://redisson.pro) ·
[License: Apache-2.0](https://github.com/redisson/redisson/blob/master/LICENSE.txt)

## Overview

Redisson is a Redis/Valkey client for the JVM, but it sits at a different abstraction layer than the other well-known Java clients. Where Jedis and Lettuce map more or less one-to-one onto Redis commands (`GET`, `SET`, `HSET`, `EVAL`), Redisson hides commands entirely behind familiar Java interfaces: `RMap` implements `java.util.concurrent.ConcurrentMap`, `RLock` implements `java.util.concurrent.locks.Lock`, `RQueue` implements `java.util.Queue`, and so on[^1]. The pitch is that a developer already fluent in the JDK concurrency and collections APIs can get a distributed, cross-JVM version of those structures with minimal new vocabulary.

The catalog is large — 50+ distributed objects and services including maps, multimaps, sets, sorted sets, queues, deques, locks (fair, multi, read-write, spin), semaphores, rate limiters, bloom/cuckoo filters, HyperLogLog, atomic counters, publish/subscribe, a remote-invocation service, a distributed `ExecutorService` and `ScheduledExecutorService`, a MapReduce service, and Live Objects (POJO-to-hash mapping via annotations)[^1]. On top of that it ships cache integrations (Spring Cache, JCache/JSR-107, Hibernate 2nd-level cache, MyBatis, Quarkus, Micronaut) and web session stores (Tomcat, Spring Session).

The defining tension is **open-core**. Redisson is Apache-2.0 and genuinely usable for free, but a meaningful set of scaling and reliability features — data partitioning of a single structure across cluster shards, advanced local-cache sync, batched/optimized operations, and performance tuning — live in the commercial **Redisson PRO** build[^2]. The community edition is not crippled, but production teams frequently hit a feature that turns out to be PRO-only. Note also that "Redis" here increasingly means Valkey too: after Redis Inc.'s 2024 license change, Redisson added first-class Valkey support and now markets itself for both.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.x.x</version>
</dependency>
```

```java
Config config = new Config();
config.useSingleServer()
      .setAddress("redis://127.0.0.1:6379");

RedissonClient redisson = Redisson.create(config);

// A distributed ConcurrentMap
RMap<String, Integer> map = redisson.getMap("scores");
map.put("tom", 42);

// A distributed lock with automatic lease renewal
RLock lock = redisson.getLock("job:nightly");
lock.lock();
try {
    // critical section, safe across JVMs
} finally {
    lock.unlock();
}

redisson.shutdown();
```

The synchronous API shown here has async (`RMapAsync`), Reactive Streams (`RMapReactive`), and RxJava3 (`RMapRx`) mirrors for every structure.

## Architecture / How It Works

**Transport.** Redisson is built on Netty for non-blocking I/O. A single client manages an asynchronous connection pool per node and understands every Redis/Valkey topology — single, cluster, sentinel, replicated, master/slave, and (in PRO) proxy and multi-cluster modes. Topology changes (failover, slot migration) are handled by reconnection and command retry rather than surfacing to the caller.

**Server-side Lua is the core mechanism.** Most of Redisson's "objects" are not single commands — an `RMap.put` that maintains an eviction index, or an `RLock.lock` that checks ownership and re-entrancy, compiles to a Lua script executed atomically on the server via `EVAL`. This is how Redisson gets atomicity for multi-step operations without client-side transactions. The consequence is that Redisson's correctness depends on those scripts running atomically, and a subset of behavior differs between standalone and cluster deployments where a script's keys must hash to the same slot.

**Locks and the watchdog.** `RLock` is the most-cited feature. When you call `lock()` without an explicit lease time, Redisson acquires the lock with a 30-second expiry and starts a background "watchdog" that renews the expiry every 10 seconds for as long as the owning JVM is alive[^3]. This prevents a crashed holder from deadlocking the lock forever while also avoiding premature expiry of a long-running holder. `RedissonMultiLock` and the Redlock-style `RedissonRedLock` coordinate a lock across independent nodes; note that plain `RLock` already works across a cluster, and the standalone RedLock class has been de-emphasized in newer releases.

**Codecs.** Every value crossing the wire is serialized by a pluggable codec — Kryo, Jackson JSON, Smile, CBOR, MsgPack, Avro, Protobuf, LZ4/Snappy wrappers, or JDK serialization. The codec is a client-wide (or per-object) setting, and the byte format it produces is what actually lives in Redis.

**Live Objects and services.** `@REntity`-annotated POJOs are transparently backed by a Redis hash, with field access translating to `HGET`/`HSET`. The distributed `ExecutorService` serializes a task (as a `Callable`/`Runnable`), pushes it to a queue, and remote worker JVMs pull and run it — a full compute-grid pattern layered on Redis.

## Production Notes

**Distributed locks are not a fencing token.** The watchdog reduces but does not eliminate the classic Redlock safety problem: a long JVM GC pause (or network partition) can exceed the lease, Redis expires the lock, another client acquires it, and the original thread resumes still believing it holds the lock. Redisson locks are fine for coordination and best-effort mutual exclusion; they are not a substitute for a fencing token where correctness on the protected resource is mandatory. Understand this before using `RLock` to guard money movement or exactly-once side effects.

**The PRO wall is real.** Data partitioning (spreading one large `RMap`/`RSet` across all cluster shards instead of pinning it to one), local cache with server-side invalidation across nodes, and several batch/pipeline optimizations are PRO-only[^2]. Teams that architect around a huge single structure in community edition can discover it lives entirely on one shard's memory. Check the feature-comparison page before committing to a design.

**Codec changes are a data-migration event.** The serialized bytes in Redis are codec-specific. Switching the default codec (or upgrading across a Redisson version that changed the default) means existing keys become unreadable by the new client unless you migrate them. Pin the codec explicitly in config rather than relying on the default.

**Connection pool sizing under load.** Redisson multiplexes commands over a bounded pool; blocking operations (`RLock`, blocking queues, pub/sub) hold connections. Under high lock churn or many blocking consumers you can starve the pool and see timeouts that look like server problems but are client-side exhaustion. `connectionPoolSize`, `subscriptionConnectionPoolSize`, and per-node minimums need tuning for lock-heavy workloads.

**Upgrades touch config.** Redisson's config schema and some defaults have shifted across 3.x minor versions, and the Spring Boot starter tracks specific Boot versions. Major jumps have historically required config edits and occasional codec/API adjustments; read the CHANGELOG before bumping.

**Cluster and cross-slot operations.** Multi-key operations require their keys to reside on the same hash slot in cluster mode. Some Redisson structures handle this transparently, others require `{hashtag}` key naming or are constrained; test cluster behavior rather than assuming standalone semantics carry over.

## When to Use / When Not

**Use when:**
- You want distributed data structures with a `java.util`-shaped API rather than hand-writing command sequences and Lua.
- You need distributed locks, semaphores, rate limiters, or a compute grid on infrastructure you already run on Redis/Valkey.
- You're in the Spring/JCache/Hibernate ecosystem and want a drop-in distributed cache with real integrations.
- You want one client that abstracts single/cluster/sentinel topologies and handles failover reconnection.

**Avoid when:**
- You want a thin, predictable command mapping with minimal magic — Lettuce or Jedis are closer to the metal and easier to reason about.
- Your locking needs true correctness guarantees on the protected resource (use a system with fencing tokens or a real consensus store).
- Your scaling plan depends on partitioning a single large structure across shards, and you're unwilling to pay for PRO.
- You want to minimize dependency surface — Redisson pulls in Netty and a codec stack; a raw client is lighter.

## Alternatives

- redis/jedis — the original synchronous Java client; thin command mapping, simplest mental model, no built-in async.
- redis/lettuce (lettuce-io/lettuce-core) — Netty-based, async/reactive from the ground up; the common choice when you want commands, not high-level objects. Spring Boot's default Redis driver.
- spring-projects/spring-data-redis — repository/template abstraction over Jedis or Lettuce; use when you want Spring Data idioms, not a distributed-objects catalog.
- hazelcast/hazelcast — an in-memory data grid (not a Redis client); use instead when you want the grid to be the datastore itself with embedded clustering.
- apache/ignite — distributed cache/compute platform; use when you need SQL over the grid and co-located compute rather than a Redis front-end.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-01 | First public commit; Redis-backed distributed Java objects[^4]. |
| 2.x | 2016 | Broad object/service catalog, cluster & sentinel support. |
| 3.0 | 2016-12 | API overhaul; async/reactive models formalized. |
| 3.x | 2018–2023 | RxJava3, JCache, Spring Boot starters, Live Objects, PRO feature split matured. |
| 3.x | 2024 | Valkey support added following Redis's license change[^5]. |

(Redisson does not use headline marketing versions; it ships frequent 3.x point releases. Consult the CHANGELOG for exact dates.)

## References

[^1]: Redisson README and feature list. https://github.com/redisson/redisson
[^2]: Redisson PRO vs. Community Edition feature comparison. https://redisson.pro/feature-comparison.html
[^3]: Redisson documentation, "Locks and synchronizers" (lock watchdog / `lockWatchdogTimeout`). https://redisson.pro/docs/data-and-services/locks-and-synchronizers
[^4]: Repository creation date from GitHub API (created 2014-01-11).
[^5]: Valkey project (fork created after Redis's 2024 relicensing). https://valkey.io

## Tags

java, redis, valkey, distributed-cache, distributed-locks, netty, jvm, redis-client, spring, open-core
