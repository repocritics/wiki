# hazelcast/hazelcast

> A distributed in-memory data grid and stream-processing engine for the JVM, run either embedded in your app or as a standalone cluster.

[GitHub repo](https://github.com/hazelcast/hazelcast) ·
[Official website](https://www.hazelcast.com) ·
[License: Apache-2.0 (dual-licensed)](https://github.com/hazelcast/hazelcast/blob/master/LICENSE)

## Overview

Hazelcast is a distributed in-memory store and compute platform for the JVM. Its core is an in-memory data grid (IMDG): distributed implementations of familiar structures — `IMap`, `IQueue`, `ISet`, `ITopic`, `IAtomicLong` — that transparently partition and replicate their data across a cluster of nodes ("members"). On top of that sits a data-processing engine, originally the separate **Jet** project, which was folded into the core in Hazelcast 5.0 (2021) to make one "unified real-time data platform" spanning caching, messaging, SQL, and streaming pipelines[^1].

The defining property is that a Hazelcast cluster is *symmetric and peer-to-peer*: there is no master node, no external coordinator (no ZooKeeper), and no separate name server. Members discover each other and form a cluster, data is partitioned across them (271 partitions by default) with configurable backups, and the cluster rebalances automatically when members join or leave. You can run it two ways: **embedded**, where the cluster members are your own JVM processes and data lives in the same heap as your application, or **client-server**, where you run a standalone Hazelcast cluster and connect thin clients (Java, Python, Node.js, .NET, C++, Go) to it.

The central tension is the split between the Apache-2.0 open-source core and the Enterprise edition. Many of the features an operator actually reaches for in production — WAN replication across data centers, disk persistence (Hot Restart / Persistence), the off-heap High-Density Memory Store, TLS/RBAC security, and blue-green client failover — are Enterprise-licensed. The open-source grid is genuinely useful, but a real deployment tends to bump into the license boundary quickly[^2].

## Getting Started

Add the dependency (Maven):

```xml
<dependency>
  <groupId>com.hazelcast</groupId>
  <artifactId>hazelcast</artifactId>
  <version>5.5.0</version>
</dependency>
```

Embedded mode — the process *is* a cluster member:

```java
import com.hazelcast.core.Hazelcast;
import com.hazelcast.core.HazelcastInstance;
import java.util.Map;

public class Main {
    public static void main(String[] args) {
        // Starts (or joins) a cluster. Run this twice → a 2-member cluster.
        HazelcastInstance hz = Hazelcast.newHazelcastInstance();

        Map<String, String> users = hz.getMap("users"); // distributed IMap
        users.put("1", "Tom");
        System.out.println(users.get("1")); // readable from any member

        hz.shutdown();
    }
}
```

Client-server mode connects to an already-running cluster instead:

```java
HazelcastInstance client = HazelcastClient.newHazelcastClient();
Map<String, String> users = client.getMap("users");
```

## Architecture / How It Works

**Partitioning.** Every distributed structure's keys are hashed into a fixed number of partitions (271 by default). Each partition has one owner member and, by default, one backup on another member. Losing a member promotes its backups and re-replicates to restore the configured backup count. This is an AP (available/partition-tolerant) design by default: reads and writes stay available during membership churn, and consistency is eventual for backups unless you configure synchronous backups.

**Discovery and clustering.** Members find each other via a pluggable discovery SPI: multicast (dev only), static TCP/IP lists, or cloud plugins for Kubernetes, AWS, GCP, Azure, Eureka, and Consul. Once joined, the oldest member acts as a lightweight cluster coordinator for partition-table decisions, but data operations are still peer-to-peer and go directly to partition owners.

**CP Subsystem.** For cases where AP eventual consistency is unacceptable, Hazelcast ships a **Raft-based CP Subsystem** (introduced in 3.12) providing linearizable `IAtomicLong`, `ILock`, `ISemaphore`, and `FencedLock`[^3]. It runs alongside the AP grid as a separate consistency domain with its own group of members, and it is explicitly opt-in and requires a configured group size.

**Jet / pipelines and SQL.** The streaming engine models computation as a DAG of vertices executed cooperatively across all members using green threads, with backpressure and at-least-once / exactly-once guarantees via distributed snapshots. The SQL engine sits on top of the same execution model, letting you query `IMap` contents and Kafka/file/JDBC sources with standard SQL.

**Serialization.** Because data crosses the network and lives off your object graph, serialization format is a first-class, load-bearing decision. Plain Java serialization works but is slow and bulky. Hazelcast offers `IdentifiedDataSerializable` and `Portable` (both requiring boilerplate factories), and the newer **Compact serialization** (GA in 5.2) which is schema-based, versionable, and does not require the class on the server side. Getting this wrong is the most common source of both performance problems and cross-version headaches.

## Production Notes

**Heap and GC are the ceiling.** In embedded mode your grid data shares the application heap; large caches mean large heaps and long GC pauses that stall both your app and cluster heartbeats. Options are client-server mode (isolate the data JVMs), aggressive tuning of a low-pause collector, or the Enterprise-only off-heap High-Density Memory Store. Sizing members past ~30–40 GB heap without off-heap storage tends to invite GC trouble.

**Split-brain is real and must be planned for.** A network partition can produce two sub-clusters that each keep operating and diverge. Hazelcast provides split-brain protection (a minimum-cluster-size quorum that fails operations below threshold) and, on heal, per-structure **merge policies** (e.g. `PutIfAbsentMergePolicy`, `LatestUpdateMergePolicy`). The defaults are permissive; if you care about correctness you must configure both quorum and merge policy explicitly.

**Serialization/version coupling.** Rolling upgrades and mixed-version clusters work, but only within supported version windows, and only if your serialization format tolerates schema evolution. `Portable`/`Compact` handle added fields gracefully; `IdentifiedDataSerializable` and Java serialization do not without manual versioning. A schema mismatch surfaces as runtime deserialization errors, not compile errors.

**Client vs member classpath.** In client-server mode, `EntryProcessor`s, predicates, `MapStore`s, and Jet pipeline lambdas execute on the *members*, so their classes must be on the member classpath (or deployed via user-code deployment). "Works embedded, fails client-server" almost always traces to a class the member JVM cannot load.

**Enterprise boundary.** Budget for the license split early. Persistence to disk, WAN replication, off-heap storage, security (TLS, mutual auth, RBAC), and the Management Center's advanced features are Enterprise. Prototyping on open-source and discovering the durability/security story is paywalled late in a project is a recurring pattern.

**Networking assumptions.** Members expect stable, low-latency connectivity and open ports between every pair (full mesh). It is not designed to span high-latency links directly — that is what WAN replication (Enterprise) is for. Kubernetes deployments need the official Helm chart / operator to get discovery and headless services right.

## When to Use / When Not

**Use when:**
- You need a distributed cache or shared in-memory state across many JVM app instances with automatic partitioning and failover.
- You want caching, pub/sub messaging, distributed locks/coordination, and stream processing from one dependency instead of stitching several together.
- You want to embed the grid directly in a JVM application (no separate infrastructure to operate).
- You need low-latency SQL over in-memory data enriched from Kafka/JDBC sources.

**Avoid when:**
- Your stack is not JVM-centric — the non-Java clients are capable but the server is Java and the ecosystem assumes it.
- You need a single-node cache or simple key-value store — Redis or Caffeine is far less operational overhead.
- You need durability, cross-DC replication, or security out of the box on a zero budget — those are Enterprise.
- You want dedicated best-in-class stream processing at scale — a purpose-built engine like Flink is more capable for complex streaming.

## Alternatives

- redis/redis — use instead when you want a battle-tested standalone data store/cache with a huge ecosystem and don't need embedded JVM co-location.
- apache/ignite — use instead when you want a similar in-memory data grid with a stronger SQL/transactional (more DB-like) posture.
- infinispan/infinispan — use instead when you're in the Red Hat/Quarkus/JBoss ecosystem and want a fully Apache-licensed grid without an Enterprise split.
- apache/flink — use instead when stream processing is the primary goal rather than caching plus compute.
- ben-manes/caffeine — use instead when you only need a fast in-process cache on a single node with no distribution.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2009 | First releases as an in-memory data grid (Talip Ozturk). |
| 3.0 | 2013-09 | Major API overhaul; long-lived 3.x line. |
| 3.12 | 2019 | Raft-based CP Subsystem introduced[^3]. |
| Jet 0.x–4.x | 2017–2020 | Separate stream-processing engine project. |
| 4.0 | 2020-02 | Networking and API modernization. |
| 5.0 | 2021-09 | IMDG + Jet unified into one platform; SQL engine[^1]. |
| 5.2 | 2022 | Compact serialization GA. |
| 5.x | 2023–2026 | Ongoing 5.x line; SQL, connectors, tiered storage, Kubernetes tooling. |

## References

[^1]: Hazelcast blog, "Hazelcast Platform 5.0" — the release that merged the IMDG and Jet engines into a single unified platform. https://hazelcast.com/blog/hazelcast-platform-5-0-ga/
[^2]: Hazelcast documentation, "Editions and Distributions" — Open Source vs Enterprise feature split. https://docs.hazelcast.com/hazelcast/latest/editions-distributions
[^3]: Hazelcast documentation, "CP Subsystem" — Raft-based strongly-consistent structures. https://docs.hazelcast.com/hazelcast/latest/cp-subsystem/cp-subsystem

## Tags

java, jvm, distributed-cache, in-memory-data-grid, stream-processing, distributed-systems, caching, low-latency, clustering, sql, pub-sub, raft
