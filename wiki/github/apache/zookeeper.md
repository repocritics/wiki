# apache/zookeeper

> A replicated in-memory tree of small nodes with ordered writes and watches — the coordination primitive a generation of distributed systems was built on.

[GitHub repo](https://github.com/apache/zookeeper) ·
[Official website](https://zookeeper.apache.org) ·
[License: Apache-2.0](https://github.com/apache/zookeeper/blob/master/LICENSE.txt)

## Overview

ZooKeeper is a coordination service for distributed applications. It exposes a small, filesystem-like namespace of nodes ("znodes") that is replicated across a cluster ("ensemble") of servers and kept consistent by a leader-based atomic broadcast protocol. Applications do not use it as a database; they use it to solve the coordination problems underneath a distributed system — leader election, group membership, distributed locks, configuration distribution, service discovery — by composing three primitives: ordered znode creation, ephemeral znodes tied to a client session, and one-time watches.

It began at Yahoo! Research and was a Hadoop subproject before graduating to a top-level Apache project in November 2010[^1]. For most of the 2010s it was the de facto coordination layer of the JVM data ecosystem: Kafka, HBase, Solr, Hadoop HDFS/YARN, Storm, and Druid all leaned on it. That ubiquity is now the defining tension. Newer systems embed their own consensus (Raft) rather than operating a separate ZooKeeper ensemble, and the most visible example — Apache Kafka — has removed its ZooKeeper dependency entirely as of Kafka 4.0[^2]. ZooKeeper is mature, stable, and widely deployed; it is also increasingly a component teams inherit rather than newly choose.

The core constraint to internalize: the entire dataset lives in memory on every server, and every write is replicated to a quorum before it commits. ZooKeeper is optimized for a high read-to-write ratio over a small amount of data. It is emphatically not a general-purpose key-value store.

## Getting Started

```bash
# Server requires a JVM (Java 8+/11+)
curl -O https://downloads.apache.org/zookeeper/zookeeper-3.9.3/apache-zookeeper-3.9.3-bin.tar.gz
tar xzf apache-zookeeper-3.9.3-bin.tar.gz
cd apache-zookeeper-3.9.3-bin
cp conf/zoo_sample.cfg conf/zoo.cfg
bin/zkServer.sh start
bin/zkCli.sh -server 127.0.0.1:2181
```

```java
// Java client — create an ephemeral, sequential znode (lock / leader-election primitive)
ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 3000, event -> {
    // Watches are ONE-TIME: re-register inside the callback if you need more.
    System.out.println("event: " + event.getType() + " " + event.getPath());
});

String path = zk.create(
    "/locks/resource-",
    new byte[0],
    ZooDefs.Ids.OPEN_ACL_UNSAFE,          // no real ACL — do not use in production
    CreateMode.EPHEMERAL_SEQUENTIAL);      // e.g. /locks/resource-0000000007
// Lowest sequence number holds the lock; watch the next-lower znode to wait.
```

For anything beyond primitive calls, use Apache Curator, which implements the correct, race-free recipes for locks, leader election, and caches[^3]. Hand-rolling these on the raw API is a well-known source of subtle bugs.

## Architecture / How It Works

An ensemble is an odd-sized set of servers (typically 3, 5, or 7). One is elected **leader**; the rest are **followers**. Writes are forwarded to the leader, which sequences them and broadcasts them via **ZAB** (ZooKeeper Atomic Broadcast) — a leader-based protocol providing totally-ordered atomic updates, distinct from Paxos though it offers comparable guarantees[^4]. A write commits once a quorum (majority) has persisted it to its transaction log. This is why a 5-node ensemble tolerates 2 failures and a 3-node ensemble tolerates 1: `floor(N/2)+1` must be reachable.

Every server holds the full data tree in memory and serves reads locally. That makes reads fast and horizontally scalable but means **reads can be stale** — a follower may lag the leader. ZooKeeper guarantees per-client FIFO ordering and linearizable writes, but a read reflects a point in time no later than the client's own last operation, not necessarily the global latest; a client needing an up-to-date view must issue `sync()` first. **Observers** are non-voting members that receive the commit stream to scale read throughput without enlarging the voting quorum (which would slow writes).

The data model is a hierarchy of znodes, each holding a small blob (default cap ~1 MB via `jute.maxbuffer`) plus metadata (version, ACL, child list, timestamps). Znode flavors: **persistent** (survive until deleted); **ephemeral** (deleted when the creating session ends — the basis of failure detection and membership); and **sequential** (the server appends a monotonic counter to the name). Ephemeral + sequential is the standard lock / leader-election building block.

Clients hold a **session** with a negotiated timeout, kept alive by heartbeats. If the client cannot heartbeat within the timeout, the session expires and its ephemeral znodes vanish — which is how the cluster learns a node is gone. **Watches** are one-time triggers registered on a znode; the server fires exactly one notification on the next change, after which the client must re-register. There is no persistent subscription in the classic API (recursive/persistent watches were added later).

## Production Notes

- **Sessions and ephemerals are a footgun during GC pauses and network blips.** A long stop-the-world pause or brief partition can expire a session, delete its ephemeral znodes, and thereby release a lock or trigger a leader re-election the application never intended. Session-timeout tuning is a real tradeoff: too short causes false expirations, too long delays genuine failure detection.
- **Watches are one-shot and can miss intermediate states.** Between a watch firing and re-registering, multiple changes can occur. Correct code always re-reads state on notification rather than trusting the event payload.
- **Herd effect.** Naively watching one "lock" znode wakes every waiter on every release. The canonical fix (each waiter watches only the next-lower sequential znode) is why Curator exists; do not reinvent it.
- **The dataset must fit in RAM, and large znodes hurt.** ZooKeeper is for kilobytes of coordination metadata, not application data. Large blobs or millions of znodes inflate snapshots, slow restarts, and can blow past `jute.maxbuffer` (which must match on clients *and* servers).
- **Disk matters more than expected.** Every write is fsynced to the transaction log before ack. Put `dataLogDir` on a dedicated low-latency device separate from snapshots; a slow log disk directly caps write throughput and can cause elections under load.
- **Even-sized ensembles are a mistake.** Four nodes tolerate the same single failure as three but carry a larger quorum. Always use odd sizes; add Observers, not voters, to scale reads.
- **Security defaults are permissive.** `OPEN_ACL_UNSAFE` and unauthenticated access leak from tutorials into production. Enable SASL/Kerberos or at least digest ACLs, and firewall the client and quorum ports.

## When to Use / When Not

**Use when:**
- You already run JVM infrastructure (Kafka pre-4.0, HBase, Solr, Hadoop) that depends on it — the majority case today.
- You need battle-tested strong-consistency coordination and can operate a stateful ensemble.
- Your coordination state is small, read-heavy, and tolerant of quorum write cost.

**Avoid when:**
- You are starting greenfield and want coordination without running a separate cluster — an embedded Raft library or etcd is lower operational cost.
- You need to store more than trivial data, or need high write throughput.
- Your stack is Kubernetes-native — etcd is already present and Consul integrates more naturally.

## Alternatives

- etcd-io/etcd — Raft-based, gRPC API, the store behind Kubernetes; prefer it for cloud-native/greenfield coordination.
- hashicorp/consul — coordination plus first-class service discovery, health checking, and multi-datacenter; use when service discovery is the actual goal.
- apache/kafka — no longer a client but a cautionary signal: KRaft mode removed the ZooKeeper dependency, endorsing embedded consensus over a separate ensemble.
- Standalone Raft libraries (e.g. apache/ratis, Hazelcast) — use when you want consensus inside your process rather than as an external service.

## History

| Version | Date | Notes |
|---------|------|-------|
| TLP | 2010-11 | Graduated from Hadoop subproject to top-level Apache project[^1]. |
| 3.4.x | 2011 | Long-lived stable line; the baseline many legacy deployments still run. |
| 3.5.x | 2019 | 3.5.5 first stable 3.5; dynamic reconfiguration, TLS, local sessions[^5]. |
| 3.6.x | 2020 | Persistent/recursive watches, expanded metrics and observability. |
| 3.7.x | 2021 | Continued protocol, security, and tooling improvements. |
| 3.8.x | 2022 | Maintenance and dependency modernization line. |
| 3.9.x | 2023 | Current stable line as of writing. |

## References

[^1]: Apache Software Foundation — ZooKeeper became a top-level project in 2010. https://zookeeper.apache.org/
[^2]: Apache Kafka — KRaft replaces ZooKeeper; ZooKeeper mode removed in Kafka 4.0 (KIP-500). https://kafka.apache.org/documentation/#kraft
[^3]: Apache Curator — high-level client with correct recipes for locks, leader election, and caches. https://curator.apache.org/
[^4]: "ZooKeeper: Wait-free coordination for Internet-scale systems," Hunt et al., USENIX ATC 2010. https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf
[^5]: ZooKeeper 3.5 reconfiguration documentation. https://zookeeper.apache.org/doc/r3.5.5/zookeeperReconfig.html

## Tags

java, distributed-systems, consensus, coordination, service-discovery, zab, apache, leader-election, key-value, configuration-management
