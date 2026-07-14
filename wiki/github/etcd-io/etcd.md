# etcd-io/etcd

> A distributed, strongly-consistent key-value store built on Raft — the coordination substrate underneath Kubernetes and most CNCF infrastructure.

[GitHub repo](https://github.com/etcd-io/etcd) ·
[Official website](https://etcd.io) ·
[License: Apache-2.0](https://github.com/etcd-io/etcd/blob/main/LICENSE)

## Overview

etcd is a distributed key-value store that prioritizes consistency and reliability over raw throughput. It uses the Raft consensus algorithm to replicate a log across a small cluster of members, presenting linearizable reads and writes over a gRPC API[^1]. It began at CoreOS in 2013 as the coordination layer for their fleet tooling, and is now a CNCF graduated project maintained under the Kubernetes umbrella (SIG-etcd)[^2].

Its defining role is as the backing store for Kubernetes: every object in a Kubernetes cluster — pods, secrets, config maps, leases — lives in etcd, and the API server is essentially a typed façade over it. This coupling drives etcd's design priorities and also its reputation: most operators encounter etcd not as a database they chose, but as a component they must keep healthy to keep Kubernetes healthy.

The central tradeoff is that etcd is deliberately *small*. It is not a general-purpose database. Consensus requires a majority quorum on every write, which caps a cluster at a handful of members (3 or 5 in practice) and bounds the dataset to gigabytes, not terabytes. The default storage quota is 2 GiB and raising it past ~8 GiB is discouraged[^3]. etcd trades scale and write throughput for the property that matters for coordination: it never loses committed data and never returns split-brain answers.

## Getting Started

```bash
# Download a release binary (Linux/macOS/Windows) from the releases page,
# or run a single-member dev cluster via Docker:
docker run -d --name etcd -p 2379:2379 \
  quay.io/coreos/etcd:v3.5.17 \
  /usr/local/bin/etcd \
  --advertise-client-urls http://0.0.0.0:2379 \
  --listen-client-urls http://0.0.0.0:2379
```

```bash
# etcdctl speaks the v3 API (set the version explicitly — it matters)
export ETCDCTL_API=3

etcdctl put /config/feature-x "enabled"
etcdctl get /config/feature-x
etcdctl get --prefix /config/          # range read over a key prefix

# Watch a key for changes (the primitive Kubernetes controllers build on)
etcdctl watch --prefix /config/
```

The client and server ports are `2379` (client) and `2380` (peer) by convention, registered with IANA[^1].

## Architecture / How It Works

**Raft consensus.** etcd's core is a Raft implementation (`go.etcd.io/raft`) that maintains a replicated log across members. One member is the leader; all writes route through it, get appended to the log, and are committed once a majority acknowledges. A read can be *serializable* (served from any member's local state, possibly stale) or *linearizable* (the default, requiring a quorum round-trip to confirm the leader is still current). Quorum is `(N/2)+1`, so a 3-member cluster tolerates 1 failure and a 5-member cluster tolerates 2. Even-sized clusters are pointless: 4 members tolerate the same single failure as 3 while needing more acks.

**MVCC storage.** Underneath, etcd keeps a multi-version concurrency-control store backed by **bbolt**, a fork of Ben Johnson's BoltDB (a single-file B+tree)[^4]. Every write creates a new revision; the store is logically append-only, indexed by a monotonically increasing global revision number. This is what makes `watch` reliable — a client can resume watching from any past revision and receive the exact sequence of changes since.

**Compaction and defragmentation are two separate things**, and conflating them is a classic operator mistake. *Compaction* discards old MVCC revisions (history) but does not shrink the file. *Defragmentation* rewrites the bbolt file to reclaim the freed space, and it blocks the member it runs on. Neither happens automatically unless configured (`--auto-compaction-mode`/`--auto-compaction-retention`).

**The v2/v3 split.** etcd 2.x and 3.x are effectively different data models. v3 introduced the flat MVCC store, gRPC API, leases, and transactions, and is not wire-compatible with v2's tree-structured JSON-over-HTTP API. This is why `ETCDCTL_API=3` exists — early v3 releases defaulted the CLI to the v2 API for compatibility. The v2 API and its store were fully removed in the 3.6 line[^5].

**Leases and watches** are the two features that make etcd a coordination primitive rather than a plain KV store. A lease is a TTL that keys can attach to; when the lease expires (or its keepalive stops), the keys are deleted — this underpins service registration and leader election. Watches stream ordered change events over a single long-lived gRPC connection.

## Production Notes

**Disk latency is the whole game.** etcd's performance is dominated by fsync latency on the WAL, not CPU or network. On slow or contended disks (network-attached volumes, shared cloud disks with burst-credit throttling) the leader cannot commit fast enough, heartbeats miss, and the cluster churns through leader elections. The consistent guidance is dedicated low-latency SSDs; the `wal_fsync_duration_seconds` and `backend_commit_duration_seconds` metrics are the first things to watch[^3].

**The database size quota is a hard wall, and hitting it is a self-inflicted outage.** When the DB exceeds `--quota-backend-bytes` (default 2 GiB), etcd raises a cluster-wide `NOSPACE` alarm and goes **read-only** until an operator compacts, defragments, *and* manually disarms the alarm. Kubernetes clusters that never configured auto-compaction routinely discover this the hard way after months of accumulated revision history.

**Odd member counts, spread across failure domains.** Run 3 or 5 members, each in a different availability zone. More members means more replication overhead and *slower* writes, not more capacity — etcd does not shard. Two-member clusters are worse than one (they can't form a quorum after either fails).

**Defrag is disruptive.** Defragmenting blocks the target member from serving requests for its duration. Do it one member at a time, leader last, ideally off-peak. Automating defrag across all members simultaneously has taken clusters offline.

**Upgrades are strictly one minor version at a time.** You cannot skip from 3.4 to 3.6; you go 3.4 → 3.5 → 3.6, and downgrade support is limited and version-specific. The 3.5 line had a notable data-inconsistency bug (fixed in 3.5.3) serious enough that the maintainers publicly recommended upgrading[^6]; it remains a cautionary tale about pinning to unpatched patch releases for something this critical.

**Auth and TLS are opt-in and easy to skip.** etcd runs with no authentication by default. Peer and client TLS, plus the RBAC auth system, must be configured explicitly. An etcd endpoint exposed without auth is a full read/write handle on (in the Kubernetes case) every secret in the cluster.

**Robustness testing.** The project runs an in-tree Jepsen-style robustness suite that injects faults and validates linearizability, which is a meaningful signal of maturity for a component in this position[^7].

## When to Use / When Not

**Use when:**
- You need distributed coordination: leader election, distributed locks, service discovery, or a source of truth for cluster configuration.
- Correctness under partition matters more than throughput — you want linearizable reads and guaranteed-durable writes.
- You're running Kubernetes (you're using etcd whether you chose it or not).
- Your working set is small (megabytes to low gigabytes) and changes are watched rather than bulk-queried.

**Avoid when:**
- You need a general-purpose database: large datasets, high write throughput, complex queries, or full-text search. etcd caps out fast.
- Your data is bigger than a few gigabytes — the quota and defrag mechanics make this painful.
- You need geo-distributed writes across high-latency links; quorum round-trips make cross-region clusters slow.
- You just want a fast cache or ephemeral store — Redis or a plain KV cache is the right tool, without the consensus overhead.

## Alternatives

- HashiCorp consul — service discovery + KV with a similar Raft core; heavier feature set (health checks, service mesh, multi-datacenter). Use instead when you want service discovery as a product, not a primitive.
- apache/zookeeper — the older coordination service (ZAB consensus); use when you're in a JVM/Hadoop/Kafka-adjacent ecosystem where it's already the standard, though etcd is generally the more modern choice.
- Redis (redis/redis) — use instead when you need a fast in-memory store/cache and can tolerate weaker durability/consistency guarantees; not a consensus system.
- cockroachdb/cockroach — use instead when you actually need a scalable, distributed *SQL* database rather than a small coordination store (CockroachDB itself uses a Raft-per-range design internally).
- FoundationDB (apple/foundationdb) — use instead for a strictly-serializable distributed KV that scales to large datasets, at the cost of a much steeper operational learning curve.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-08 | Initial release at CoreOS; v2 tree/HTTP model. |
| 2.0 | 2015-02 | Stabilized v2 API and Raft implementation.[^1] |
| 3.0 | 2016-06 | New MVCC store, gRPC API, leases, transactions.[^5] |
| 3.2 | 2017-06 | gRPC proxy, JWT auth, watch improvements. |
| 3.4 | 2019-08 | Learner (non-voting) members, pre-vote, concurrent read improvements. |
| 3.5 | 2021-06 | Structured logging, bbolt improvements; 3.5.0–3.5.2 hit a data-consistency bug fixed in 3.5.3.[^6] |
| 3.6 | 2025 | v2 store fully removed; downgrade support, feature-gate framework.[^5] |

## References

[^1]: etcd README and project overview. https://github.com/etcd-io/etcd
[^2]: CNCF, etcd graduated project. https://www.cncf.io/projects/etcd/
[^3]: etcd docs, "Hardware recommendations" and "Maintenance". https://etcd.io/docs/latest/op-guide/hardware/
[^4]: bbolt — embedded B+tree store used as etcd's storage backend. https://github.com/etcd-io/bbolt
[^5]: etcd docs, "etcd3 API" and 3.6 upgrade notes. https://etcd.io/docs/latest/learning/api/
[^6]: etcd maintainers, data-inconsistency issue and 3.5.3 fix. https://github.com/etcd-io/etcd/issues/13766
[^7]: etcd robustness testing suite. https://github.com/etcd-io/etcd/tree/main/tests/robustness

## Tags

go, distributed-systems, key-value-store, raft, consensus, kubernetes, cncf, coordination, database, strong-consistency
