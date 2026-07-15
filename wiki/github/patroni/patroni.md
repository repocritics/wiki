# patroni/patroni

> A Python template for PostgreSQL high availability that leans on an external consensus store (etcd, Consul, ZooKeeper, Kubernetes, or Raft) to run leader election and automatic failover.

[GitHub repo](https://github.com/patroni/patroni) ·
[Documentation](https://patroni.readthedocs.io) ·
[License: MIT](https://github.com/patroni/patroni/blob/master/LICENSE)

## Overview

Patroni is an agent you run alongside each PostgreSQL instance in a cluster. Every agent competes to hold a short-lived leader key in a distributed configuration store (DCS); whichever agent holds it runs the primary, and the rest configure their local Postgres as streaming replicas of that primary. When the leader key expires and cannot be renewed — because the primary died, was partitioned, or lost DCS access — the surviving agents run a new election and one promotes its replica. It began as a fork of Compose's Governor and was developed at Zalando; the project moved from the `zalando/patroni` namespace to its own `patroni/patroni` organization[^1].

The defining characteristic — and the thing that trips up first-time operators — is that Patroni does not implement consensus itself. It borrows it. The DCS is the source of truth for who the leader is, so Patroni's availability is bounded by the DCS's availability. This is deliberate: correctly implementing distributed consensus is exactly the mistake most homegrown failover scripts make, and Patroni refuses to. The README is blunt that Patroni is "a template, not a one-size-fits-all" system and "will have its own caveats. Use wisely."[^2]

It targets DBAs, SREs, and DevOps engineers running self-managed Postgres on VMs or Kubernetes. It underpins Zalando's Postgres Operator and Spilo, and is the failover engine inside several other Postgres distributions. Supported PostgreSQL versions run from 9.3 through 18[^2].

## Getting Started

```bash
pip install 'patroni[etcd3,psycopg3]'   # DCS driver + psycopg extra are required
```

```yaml
# postgres0.yml — a single node; run one config per host
scope: demo-cluster
name: node0
restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.10:8008
etcd3:
  hosts: 10.0.0.10:2379,10.0.0.11:2379,10.0.0.12:2379
bootstrap:
  dcs:
    ttl: 30            # leader key lifetime
    loop_wait: 10      # HA loop interval
    retry_timeout: 10  # DCS/PG operation timeout
    postgresql:
      use_pg_rewind: true
postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.10:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    superuser: {username: postgres, password: secret}
    replication: {username: replicator, password: secret}
```

```bash
patroni postgres0.yml               # start the agent (it initdb's the first node)
patronictl -c postgres0.yml list    # inspect cluster / leader
patronictl -c postgres0.yml switchover   # planned, controlled role change
```

Point applications at the cluster through HAProxy (Patroni ships an example config) or a template'd proxy that reads the REST API's `/primary` and `/replica` health endpoints, rather than hardcoding a host.

## Architecture / How It Works

The core is a control loop (`loop_wait` seconds) on every node. Each iteration: read cluster state from the DCS, observe the local Postgres, and reconcile. The primary node renews the leader key with a `ttl`; if it fails to renew within `ttl`, the key expires and the node demotes itself. Replicas that see an expired key run the election. The critical invariant is `ttl > loop_wait + 2 * retry_timeout`, otherwise a slow DCS round-trip can expire a healthy leader and cause needless failover.

Pluggable DCS backends — etcd (v2 and v3), Consul, ZooKeeper, Exhibitor, Kubernetes endpoints/configmaps, and a built-in Raft implementation (`pysyncobj`) — all sit behind one interface. On Kubernetes, the DCS is the Kubernetes API itself, so no separate etcd is needed.

Each agent exposes a REST API (default 8008) used for three things: health checks that proxies key off, `patronictl` control operations, and inter-node communication during the leader race and failsafe checks. Replication is Postgres's own streaming replication, async by default; `maximum_lag_on_failover` caps how far behind a replica may be and still be promoted. `synchronous_mode` switches to synchronous replication and records the current sync standby in the DCS so failover never promotes a node that was not caught up.

To prevent split-brain when a demoted primary is slow to notice it lost the race, Patroni integrates with a hardware/software **watchdog** (typically `softdog`): the primary must keep petting the watchdog, and if the HA loop stalls the watchdog reboots the box before a second primary can accept writes. Reattaching a former primary uses `pg_rewind` when enabled; otherwise the stale node must be reinitialized from a fresh base backup.

## Production Notes

- **A DCS outage can take down writes.** If the DCS loses quorum, Patroni cannot confirm it still holds leadership, so it demotes the primary to read-only rather than risk split-brain. This surprises teams who assumed the database was independent of etcd. **Failsafe mode** (`failsafe_mode`) mitigates this by letting the primary stay up if it can still reach all other members directly, but it must be enabled explicitly.
- **Watchdog is not optional in practice.** Without it, a partitioned-but-alive old primary can keep serving writes for a window. Configure `softdog` (or a hardware watchdog) on every node that can run the primary.
- **Async failover loses data by design.** Any transaction committed on the old primary but not yet streamed is gone. Use `synchronous_mode` for zero-data-loss failover; `synchronous_mode_strict` will refuse writes when no sync standby exists (availability traded for durability).
- **Never bypass Patroni.** Do not `pg_ctl`/`systemctl` Postgres directly or edit `postgresql.conf` out of band — Patroni owns the process and config, and will revert or fight you. Use `patronictl` for switchover/failover/restart/reinit and `edit-config` for dynamic settings stored in the DCS.
- **Major version upgrades are not orchestrated.** Patroni does not drive `pg_upgrade`; you script the upgrade and pause/resume the HA loop around it.
- **Memory-restricted hosts on Python 3.11+.** Under `vm.overcommit_memory=2`, a CPython thread-start bug could hang the REST API while Postgres kept running. Releases 4.1.1+/4.0.8+ start threads early to reduce exposure; also set `MALLOC_ARENA_MAX=1` and tune `thread_stack_size`/`thread_pool_size`[^2].
- **Connect apps as a non-superuser.** Superuser sessions can exhaust `superuser_reserved_connections`, locking Patroni out of the database and producing undesirable behavior[^2].

## When to Use / When Not

**Use when:**
- You self-manage PostgreSQL and need automatic failover with a well-audited election mechanism instead of hand-rolled scripts.
- You already run (or can run) a reliable etcd/Consul/ZooKeeper cluster, or you're on Kubernetes.
- You want fine control over replication durability, bootstrap method (`initdb`, pgBackRest, WAL-G, custom), and callbacks.

**Avoid when:**
- You cannot operate a quorum DCS reliably — a flaky etcd makes Patroni worse than no automation. (The built-in Raft mode reduces moving parts but is less battle-tested than etcd.)
- You want a managed database — RDS, Cloud SQL, and Aurora already solve HA; Patroni is for the self-hosted case.
- You need orchestrated major-version upgrades or a turnkey connection proxy out of the box.

## Alternatives

- sorintlab/stolon — Go-based PG HA with a built-in connection proxy; use it when you want the proxy bundled, but note maintenance has slowed considerably.
- cloudnative-pg/cloudnative-pg — Kubernetes-native operator with no external DCS (uses the API server and its own reconciler); use it when you're all-in on Kubernetes and want operator semantics.
- EnterpriseDB/repmgr — lighter replication management with manual/scripted switchover; use it when you want failover under human control rather than fully automatic.
- hydradatabase/pg_auto_failover — a monitor-plus-keeper design with simpler topology; use it when standing up a full DCS feels like too much operational surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-07 | Repository created; fork of Compose's Governor[^1]. |
| 1.0 | 2016 | First tagged releases; Zalando adoption, KubeCon talks. |
| 2.0.0 | 2020-09 | Dropped Python 2; config and API cleanups[^3]. |
| 2.1.x | 2021 | Failsafe mode for DCS outages introduced. |
| 3.0.0 | 2023-02 | Citus integration; quorum-based synchronous improvements[^3]. |
| 4.0.0 | 2024-09 | Dropped older Python versions; dependency/API modernization[^3]. |
| 4.0.8 / 4.1.1 | 2025 | Mitigation for CPython 3.11+ thread-start hang under memory pressure[^2]. |

## References

[^1]: Patroni README, "How Patroni Works" — origin as a fork of Compose's Governor, developed at Zalando. https://github.com/patroni/patroni
[^2]: Patroni README — "template, not plug-and-play," supported PostgreSQL 9.3–18, memory-restricted Python 3.11+ guidance, superuser advice. https://github.com/patroni/patroni/blob/master/README.rst
[^3]: Patroni releases and changelog. https://github.com/patroni/patroni/releases

## Tags

postgresql, high-availability, failover, python, etcd, consul, zookeeper, kubernetes, replication, database, raft, dcs
