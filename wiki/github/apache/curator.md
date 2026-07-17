# apache/curator

> A Java client library for Apache ZooKeeper — connection management, retry logic, and battle-tested recipes for distributed coordination.

[GitHub repo](https://github.com/apache/curator) ·
[Official website](https://curator.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/curator/blob/master/LICENSE)

## Overview

Apache Curator is a high-level Java/JVM client for [Apache ZooKeeper](https://zookeeper.apache.org/), the distributed coordination service. The raw ZooKeeper client is notoriously hard to use correctly: connection loss, session expiration, watch re-registration, and retry semantics are all left to the caller. Curator wraps that client with automatic connection management, pluggable retry policies, and a library of pre-built "recipes" for common coordination problems — leader election, distributed locks, shared counters, path caches, and barriers[^1].

Curator originated at Netflix, authored primarily by Jordan Zimmerman, and was donated to the Apache Software Foundation. It entered the Apache Incubator in 2011 and graduated to a Top-Level Project in 2013[^2]. It is the de facto standard way to talk to ZooKeeper from the JVM, and sits underneath a large amount of infrastructure — Apache Hadoop ecosystem components, Apache Kafka (historically), Spring Cloud Zookeeper, and countless in-house service-discovery layers.

The defining tension is that Curator is only as relevant as ZooKeeper itself. As systems migrate off ZooKeeper — Kafka's KRaft mode removed the ZooKeeper dependency entirely, and newer designs reach for etcd or Raft-embedded consensus — Curator's addressable surface shrinks. It remains essential where ZooKeeper is already deployed, and largely irrelevant to greenfield systems that never adopt ZooKeeper in the first place.

## Getting Started

Curator is distributed via Maven Central under `org.apache.curator`. Most projects depend on `curator-recipes`, which transitively pulls in `curator-framework` and `curator-client`:

```xml
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>5.7.1</version>
</dependency>
```

```java
// Connect with an exponential-backoff retry policy, then take a distributed lock.
RetryPolicy retry = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.newClient("zk1:2181,zk2:2181", retry);
client.start();

InterProcessMutex lock = new InterProcessMutex(client, "/locks/my-resource");
if (lock.acquire(10, TimeUnit.SECONDS)) {
    try {
        // critical section — only one JVM across the cluster runs this at a time
    } finally {
        lock.release();
    }
}
```

## Architecture / How It Works

Curator is layered into distinct modules, and understanding the split matters for both dependency hygiene and debugging:

- **`curator-client`** — a thin wrapper over the raw ZooKeeper client. It owns the connection: retry policies (`ExponentialBackoffRetry`, `RetryNTimes`, `RetryForever`, `RetryUntilElapsed`), ensemble tracking, and the `ConnectionStateManager` that translates ZooKeeper's low-level events into a clean state machine.
- **`curator-framework`** — the fluent, high-level API (`client.create().creatingParentsIfNeeded().forPath(...)`). Handles the hard parts of retries: idempotent operation replay, guaranteed deletes, and namespace scoping.
- **`curator-recipes`** — the coordination primitives built on the framework: `InterProcessMutex`, `InterProcessReadWriteLock`, `InterProcessSemaphoreV2`, `LeaderLatch`, `LeaderSelector`, `SharedCount`, `DistributedAtomicLong`, `CuratorCache`, and barrier/queue recipes.
- **`curator-x-discovery`** — a service-discovery layer (register instances under a path, query by service name) with an optional REST server (`curator-x-discovery-server`).
- **`curator-x-async`** — a Java 8 non-blocking DSL returning `CompletionStage`, for callers who want async rather than the blocking framework API.

The central abstraction is **connection state**. ZooKeeper distinguishes transient connection loss from session expiration, and Curator surfaces this via `ConnectionState`: `CONNECTED`, `SUSPENDED` (connection lost, session may still be alive), `LOST` (session expired — all ephemeral nodes and locks are gone), `RECONNECTED`, and `READ_ONLY`. Every recipe's correctness hinges on how it reacts to these transitions. A `LeaderLatch`, for example, must relinquish leadership when the session is `LOST`, because another node may have already claimed it.

Recipes are implemented on ZooKeeper's core guarantees: sequential ephemeral znodes (for lock ordering and leader election), watches (for change notification), and the total ordering of the `zxid` transaction counter. Curator hides the znode bookkeeping but does not — and cannot — hide the underlying consistency model.

## Production Notes

**Connection state handling is not optional.** The single most common Curator bug is ignoring `SUSPENDED`/`LOST`. If you hold an `InterProcessMutex` and the session is lost, you no longer hold the lock — but your code keeps running unless you registered a `ConnectionStateListener` and stopped work on suspension. Recipes throw or fire listeners, but the application must act on them.

**Locks are not fencing-token-safe by default.** A JVM GC pause longer than the session timeout can cause a client to believe it still holds a lock after ZooKeeper has already expired its session and granted the lock elsewhere. This is the classic distributed-lock hazard described by Martin Kleppmann[^3]. ZooKeeper's monotonic `zxid` can serve as a fencing token, but Curator's lock API does not hand you one automatically — for correctness against a shared resource you must add fencing at the resource layer yourself.

**ZooKeeper version alignment.** Curator bundles a specific ZooKeeper client dependency, and client/server version skew causes subtle failures. Curator 5.x requires ZooKeeper 3.5.x or later and Java 8+; it removed the "zk34 compatibility mode" that Curator 4.x offered for talking to legacy ZooKeeper 3.4 ensembles[^4]. Upgrading Curator across the 4→5 boundary often forces a coordinated ZooKeeper upgrade.

**Deprecated caches.** `PathChildrenCache`, `NodeCache`, and `TreeCache` were consolidated into a single `CuratorCache` in the 5.x line. New code should use `CuratorCache`; the older caches remain but are deprecated and have known edge cases around reconnection.

**Avoid the queue recipes.** Curator ships `DistributedQueue`/`DistributedPriorityQueue`, but the project's own documentation discourages using ZooKeeper as a message queue — every operation is a synchronous write to a replicated log, throughput is low, and it does not degrade gracefully under load. Use a real broker.

**Session timeout tuning.** Too short and normal GC or network blips trigger spurious `LOST` events and leadership churn; too long and genuine failures take longer to detect. The session timeout is a floor negotiated with the server (`maxSessionTimeout`), so client-side settings can be silently clamped.

## When to Use / When Not

**Use when:**
- You already run ZooKeeper and need leader election, distributed locks, or shared counters from JVM services.
- You want service discovery on top of an existing ZooKeeper ensemble without adopting a new dependency.
- You are integrating with the Hadoop/Kafka-era ecosystem where ZooKeeper is already the coordination substrate.

**Avoid when:**
- You are building greenfield and have no ZooKeeper — adopting ZooKeeper + Curator solely for a lock is heavy operational weight.
- You need strong fencing guarantees for a shared external resource and cannot add your own fencing tokens.
- Your coordination is really a queue or high-throughput event stream — use a broker, not ZooKeeper recipes.
- Your stack is non-JVM; Curator is Java-only (other languages use native ZooKeeper bindings or different coordination tools).

## Alternatives

- apache/zookeeper — the underlying service's raw client; use it directly when you want full control and no recipe abstraction, accepting that you re-implement retries and connection handling yourself.
- etcd-io/etcd — use instead when starting fresh and you want a Raft-based key-value/coordination store; the `jetcd` client covers locks and leader election without a JVM-centric server.
- hashicorp/consul — use when service discovery, health checking, and multi-datacenter are the primary need rather than low-level ZooKeeper primitives.
- redisson — use for distributed locks and collections backed by Redis when Redis is already deployed and you accept its weaker consistency guarantees for locking.

## History

| Version | Date | Notes |
|---------|------|-------|
| Incubation | 2011 | Donated to Apache by Netflix (author Jordan Zimmerman)[^2]. |
| TLP | 2013-09 | Graduated to Apache Top-Level Project[^2]. |
| 2.x | 2013 | Apache line targeting ZooKeeper 3.4. |
| 3.0 | 2016 | Support for ZooKeeper 3.5 features (beta). |
| 4.0 | 2018 | ZooKeeper 3.5 primary, soft-compatibility ("zk34") with 3.4. |
| 5.0 | 2020-07 | Requires Java 8 and ZooKeeper 3.5+; dropped 3.4 compatibility; `CuratorCache` introduced[^4]. |
| 5.7.x | 2024 | Current 5.x maintenance line on Maven Central. |

## References

[^1]: Apache Curator project site — overview and recipe catalog. https://curator.apache.org/
[^2]: Apache Curator — project history and Netflix origin. https://curator.apache.org/docs/about
[^3]: Martin Kleppmann, "How to do distributed locking" — 2016. https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
[^4]: Apache Curator — ZooKeeper compatibility and version requirements. https://curator.apache.org/docs/zk-compatibility

## Tags

java, zookeeper, distributed-coordination, leader-election, distributed-locks, service-discovery, jvm, consensus, apache
