# ben-manes/caffeine

> A high-performance, near-optimal in-memory caching library for Java 11+, built around the W-TinyLFU eviction policy.

[GitHub repo](https://github.com/ben-manes/caffeine) ·
[User's Guide](https://github.com/ben-manes/caffeine/wiki) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0.html)

## Overview

Caffeine is an on-heap, in-process caching library for the JVM, written by Ben Manes as the successor to two earlier projects he worked on: Google Guava's `Cache` (which he co-designed) and `ConcurrentLinkedHashMap`[^1]. Its API deliberately mirrors Guava's `CacheBuilder`, so migration is mostly mechanical, but the internals are a ground-up rewrite aimed at higher throughput and a better hit rate.

The defining feature is the eviction policy. Where most caches use plain LRU (or a segmented approximation), Caffeine uses **W-TinyLFU**: a small admission window plus a frequency-based admission filter backed by a Count-Min sketch[^2]. New entries must earn their place — an arriving item is only admitted to the main region if its estimated frequency exceeds that of the eviction victim it would displace. This resists one-hit-wonder pollution (scans, bursts) that degrades LRU, and in published simulations it achieves hit rates close to the theoretical optimum across a wide range of traces[^2][^3].

Caffeine is a single-node cache: everything lives on the JVM heap, backed by a `ConcurrentHashMap`. It has no distribution, persistence, or off-heap tier of its own. That narrow scope is the whole tradeoff — it is the fast L1 in front of a database or a distributed cache, not a replacement for Redis or Ehcache's tiered storage. It is the default cache implementation wired into Spring, Micronaut, Quarkus, and many other frameworks[^4].

## Getting Started

Add the dependency (Java 11+ uses the `3.x` line; Java 8 is stuck on `2.x`):

```gradle
implementation("com.github.ben-manes.caffeine:caffeine:3.2.4")
```

```java
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .refreshAfterWrite(Duration.ofMinutes(1))
    .recordStats()
    .build(key -> createExpensiveGraph(key));   // CacheLoader

Graph g = graphs.get(key);   // loads + caches on miss, single-flight per key
```

For non-blocking loads, `AsyncLoadingCache` returns `CompletableFuture` values and populates the cache when the future resolves:

```java
AsyncLoadingCache<Key, Graph> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .buildAsync(key -> fetchGraphAsync(key));   // returns CompletableFuture<Graph>
```

## Architecture / How It Works

**Storage.** Entries live in a `ConcurrentHashMap`. Caffeine layers eviction/expiration metadata on top via a set of generated node classes: a code generator emits specialized entry types for each combination of features (strong/weak keys, expiry variants, weight) so unused fields cost no memory[^1]. This is why the jar ships hundreds of `*Node` classes.

**Read/write recording.** Recording an access on every read would serialize threads on shared LRU state. Instead, reads are appended to striped, lossy ring buffers; a single thread drains them under a `tryLock`, and if the buffer is full the record is simply dropped (the policy tolerates approximate ordering). Writes go through a bounded MPSC buffer that is always drained. Eviction and expiration bookkeeping therefore happen asynchronously and amortized, off the hot path.

**W-TinyLFU eviction.** The keyspace is split into an **admission window** (a small LRU, roughly 1% by default) and a **main region** managed as a segmented LRU (protected + probationary). When the window overflows, its victim is compared against the main region's victim using a **TinyLFU** frequency estimate — a 4-bit Count-Min sketch that is periodically halved ("aging") so the cache adapts to shifting workloads[^2]. Later versions add an **adaptive** step that resizes the window via hill-climbing to tune between recency- and frequency-biased workloads at runtime[^3].

**Expiration.** `expireAfterWrite` / `expireAfterAccess` use ordered queues; variable per-entry expiry (`expireAfter(Expiry)`) is implemented with a **hierarchical timer wheel** for O(1) scheduling. Time comes from `System.nanoTime()` by default (overridable via `Ticker` for testing).

**Refresh vs. expire.** `refreshAfterWrite` reloads an entry asynchronously *after* it becomes stale while still serving the old value; `expireAfterWrite` removes it and forces the next caller to block on a load. They are frequently combined (refresh window shorter than expire) to keep hot keys warm without stampedes.

## Production Notes

- **On-heap only.** A large Caffeine cache is a large live heap. Multi-GB caches create GC pressure and can lengthen pauses; if the working set is larger than you want to keep on-heap, that is Ehcache/off-heap or Redis territory, not Caffeine. Size by weight (`maximumWeight` + `Weigher`) when entries vary in cost.
- **`maximumSize` is approximate, not a hard cap.** Because eviction is drained asynchronously, the cache can momentarily exceed the bound before the maintenance pass trims it. Do not rely on the map size as a strict resource limit.
- **Weak/soft references are GC-driven and non-deterministic.** `weakValues()`/`softValues()` evict on the garbage collector's schedule, not the cache's, and require identity (`==`) semantics for keys with `weakKeys()`. Soft references in particular interact badly with aggressive heaps and can be cleared en masse under memory pressure.
- **Async executor default is a footgun.** `AsyncCache`, `refreshAfterWrite`, and the (asynchronous) `removalListener` run on `ForkJoinPool.commonPool()` unless you pass an `Executor`. Under load, sharing the common pool with parallel streams can starve refreshes; supply a dedicated executor in latency-sensitive services.
- **`removalListener` vs `evictionListener`.** The removal listener runs asynchronously on the executor and may lag; `evictionListener` (added in 3.x) runs synchronously during eviction under the map's computation. Use the latter only for cheap, non-blocking work — it holds cache internals.
- **Stats are not free but cheap.** `recordStats()` uses striped counters; the overhead is small, but it is off by default and must be enabled explicitly to get hit-rate/eviction metrics.
- **2.x → 3.x migration.** The only hard break is the Java baseline (8 → 11). The `Caffeine` builder API is otherwise stable across the two lines; most upgrades are a version bump. The 2.x line still receives fixes for Java 8 users.

## When to Use / When Not

**Use when:**
- You need a fast, in-process L1 cache in front of a slower store (DB, RPC, remote cache).
- Hit rate matters and your access pattern includes scans or bursts that punish LRU.
- You want single-flight loading, async loading, refresh-ahead, and per-entry expiry without hand-rolling them.

**Avoid when:**
- You need a cache shared across processes or nodes — use Redis or a distributed grid; Caffeine can sit in front of it, but is not it.
- Your dataset exceeds comfortable heap size — an off-heap/tiered cache avoids the GC cost.
- You need persistence or survival across restarts — Caffeine is purely in-memory.

## Alternatives

- google/guava — the predecessor `Cache`; simpler segmented-LRU eviction, lower hit rate and throughput. Use it when you already depend on Guava and the cache is not hot.
- cache2k/cache2k — another high-performance on-heap Java cache with an advanced eviction policy; comparable niche, different API. Use when you prefer its feature set or benchmarks for your workload.
- ehcache/ehcache3 — supports off-heap and disk tiers plus clustering. Use when the working set is larger than heap or you need persistence/tiering.
- redisson/redisson — a Redis client with distributed cache/near-cache support. Use when the cache must be shared across nodes.
- apache/ignite — distributed in-memory data grid. Use when you need a clustered cache with compute and SQL, not a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016 | First stable release; Guava-compatible builder API, W-TinyLFU eviction[^1]. |
| 2.0.0 | 2017 | Java 8 baseline; async cache, variable expiry additions over the 2.x line. |
| 3.0.0 | 2021 | Java 11 required; `evictionListener`, adaptive eviction refinements[^5]. |
| 3.2.4 | 2026 | Current release on the 3.x line[^5]. |

## References

[^1]: Caffeine README and User's Guide, ben-manes/caffeine. https://github.com/ben-manes/caffeine/wiki
[^2]: Gil Einziger, Roy Friedman, Ben Manes, "TinyLFU: A Highly Efficient Cache Admission Policy," ACM TOS. https://dl.acm.org/doi/10.1145/3149371
[^3]: Gil Einziger, Ohad Eytan, Roy Friedman, Ben Manes, "Adaptive Software Cache Management," Middleware 2018. https://dl.acm.org/doi/10.1145/3274808.3274816
[^4]: Spring Framework caching — Caffeine store configuration. https://docs.spring.io/spring-framework/reference/integration/cache/store-configuration.html#cache-store-configuration-caffeine
[^5]: Caffeine release notes. https://github.com/ben-manes/caffeine/releases

## Tags

java, jvm, caching, in-memory-cache, tinylfu, eviction-policy, lru, concurrency, library, performance, guava
