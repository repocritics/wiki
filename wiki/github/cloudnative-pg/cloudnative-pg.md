# cloudnative-pg/cloudnative-pg

> A Kubernetes operator that runs PostgreSQL primary/standby clusters using the Kubernetes API itself as the failover coordinator — no Patroni, etcd, or external DCS.

[GitHub repo](https://github.com/cloudnative-pg/cloudnative-pg) ·
[Official website](https://cloudnative-pg.io) ·
[License: Apache-2.0](https://github.com/cloudnative-pg/cloudnative-pg/blob/main/LICENSE)

## Overview

CloudNativePG (CNPG) is a Kubernetes operator, written in Go, that manages the
full lifecycle of PostgreSQL clusters — provisioning, streaming replication,
failover, backup, and rolling upgrades — as native Kubernetes resources. It was
built and open-sourced by EDB (EnterpriseDB) in April 2022, carrying design
lineage from the 2ndQuadrant Postgres team, and is led by Gabriele Bartolini[^1].
The project is a CNCF Sandbox project[^2] and one of the more actively developed
of the several competing Postgres-on-Kubernetes operators.

Its defining architectural decision — and its central tradeoff — is that it does
*not* use a separate high-availability stack. Where Zalando's operator wraps
Patroni (which needs etcd/Consul/Kubernetes as a distributed consensus store) and
older tooling used repmgr or Stolon, CNPG treats the Kubernetes API server as the
single source of truth. The operator reconciles a `Cluster` custom resource,
elects primaries, and updates Service endpoints directly. This collapses the
moving parts, but it also means the Kubernetes control plane and the operator pod
are now on the critical path for failover: if they are unavailable, automated
promotion does not happen[^3].

CNPG targets DBAs and platform teams who want Postgres to live inside a
GitOps/Kubernetes workflow rather than a managed DBaaS. It is deliberately
narrow: vanilla Kubernetes only, vanilla PostgreSQL only (no MySQL/MariaDB, no
forks unless expressible as an extension), and not a general-purpose database
operator.

## Getting Started

```bash
# Install the operator (manifest for a specific version)
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.0.yaml

# Optional: the kubectl plugin for day-2 operations
kubectl krew install cnpg
```

```yaml
# cluster.yaml — a 3-instance HA cluster (1 primary, 2 streaming replicas)
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-prod
spec:
  instances: 3
  storage:
    size: 20Gi
    storageClass: fast-ssd        # local or low-latency network storage
  postgresql:
    parameters:
      shared_buffers: "512MB"
  bootstrap:
    initdb:
      database: app
      owner: app
```

```bash
kubectl apply -f cluster.yaml
kubectl cnpg status pg-prod        # health, replication lag, elected primary
# Connect via the read-write service: pg-prod-rw.<namespace>.svc:5432
```

## Architecture / How It Works

Each PostgreSQL instance runs in its own pod backed by a PersistentVolumeClaim.
Inside the pod, an **instance manager** process runs as PID 1 and supervises the
local `postgres` process, handles signals, exposes health probes, and streams
status back to the operator. Application containers are **immutable**: an upgrade
does not mutate a running container, it rolls out a new image[^4].

The operator maintains three Services per cluster: `-rw` (routes to the current
primary), `-ro` (read-only, replicas), and `-r` (any instance). Failover is a
reconciliation: when the primary's pod/health fails, the operator picks the most
advanced standby, promotes it, and repoints the `-rw` endpoint. Rolling updates
apply to replicas first, then perform a controlled switchover of the primary.

Replication is native PostgreSQL streaming replication, configurable as
asynchronous or synchronous (including quorum-based synchronous sets). There is
no external consensus daemon — the operator's view of the world *is* the
Kubernetes API, which is why split-brain protection is ultimately bounded by
Kubernetes API consistency rather than a Postgres-specific quorum protocol.

Beyond `Cluster`, the CRD surface includes `Backup`, `ScheduledBackup`,
`Pooler` (PgBouncer connection pooling), `Database`, `Publication`,
`Subscription`, `ImageCatalog`, and `ClusterImageCatalog`. Backups historically
used the in-tree Barman Cloud integration to archive WAL and base backups to S3,
GCS, or Azure Blob; volume-snapshot-based backups were added later. As of the
1.26 line (2025), object-store backup is being externalized into the **Barman
Cloud Plugin** built on the CNPG-I plugin interface, and the in-tree path is on a
deprecation track — new deployments should plan around the plugin[^5].

## Production Notes

**The operator and control plane are on the failover path.** With no external
DCS, an unhealthy operator or an unreachable Kubernetes API means no automated
promotion. Run the operator with adequate replicas/PDBs and treat control-plane
availability as a hard dependency, not an afterthought[^3].

**Storage choice dominates performance.** Postgres is fsync-heavy. Slow or
high-latency network storage produces poor commit latency regardless of CPU. The
project's own guidance leans toward local NVMe or storage with strong latency
characteristics, and separating WAL onto its own volume (`walStorage`) is a
common tuning step. Your `storageClass` decision is effectively a database
performance decision.

**Synchronous replication can stall writes.** Quorum-based synchronous commit
protects against data loss, but if the required number of synchronous standbys
becomes unavailable, primary writes block by design. Understand the difference
between the durability you asked for and the availability you gave up before
enabling it in production.

**Major-version upgrades are the historical weak spot.** For most of the
project's life, a PostgreSQL *major* upgrade meant logical replication or
dump/restore into a new cluster rather than an in-place bump; declarative
major-version upgrade tooling arrived only relatively recently and should be
validated on non-production data before you rely on it. Minor-version bumps are
routine rolling updates via image catalogs.

**Superuser is disabled by default**, and the recommended topology is one
Postgres cluster per database/workload rather than many databases in one cluster.
Cross-region DR uses **replica clusters** (a designated primary replicating from
an external source) rather than a single stretched cluster. Backup retention,
WAL archiving credentials, and restore drills all need to be set up and *tested* —
a `ScheduledBackup` that has never been restored is not a backup.

## When to Use / When Not

**Use when:**
- You already run workloads on vanilla Kubernetes and want Postgres managed the same GitOps way.
- You want HA/failover without operating a separate Patroni + etcd/Consul stack.
- You need declarative backups to object storage and reproducible cluster definitions in Git.
- You want to avoid a proprietary DBaaS and keep data on your own infrastructure.

**Avoid when:**
- You are not on Kubernetes, or run a managed platform where a DBaaS (RDS, Cloud SQL, AlloyDB) is cheaper in total operator-hours.
- You need a PostgreSQL fork's features (Citus, Greenplum) or a non-Postgres database.
- Your team lacks Kubernetes storage/networking depth — the failure modes are Kubernetes failure modes.
- You require mature, no-surprises in-place major-version upgrades as a first-class, long-proven feature.

## Alternatives

- zalando/postgres-operator — Patroni-based operator; battle-tested at Zalando scale, but carries the external-DCS moving parts CNPG avoids. Use it when you already trust Patroni's HA semantics.
- CrunchyData/postgres-operator — mature commercial-backed operator (PGO) with a broad feature set. Use it when you want vendor support from Crunchy and its tooling ecosystem.
- percona/percona-postgresql-operator — Percona-supported, also builds on Patroni. Use it when you want Percona's cross-database support and monitoring stack.
- ongres/stackgres — opinionated Postgres platform with a web UI and bundled extensions. Use it when you want more batteries-included tooling than a bare operator.
- zalando/patroni — the HA framework itself, not an operator. Use it when you want HA on VMs/bare metal without Kubernetes.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.15.0 | 2022-04-21 | First public release; EDB open-sources the operator[^1]. |
| 1.16.0 | 2022-07-07 | Early hardening of the operator/CRD surface. |
| 1.18.0 | 2022-11-10 | Continued backup/replication feature growth. |
| 1.20.0 | 2023-04-27 | Volume-snapshot backups, expanded declarative features. |
| 1.22.0 | 2023-12-21 | Ongoing HA and backup refinements. |
| 1.24.0 | 2024-08-22 | Image catalogs and lifecycle improvements. |
| 1.26.0 | 2025-05-23 | CNPG-I plugin direction; Barman Cloud moving to a plugin[^5]. |
| 1.30.0 | 2026-06-29 | Latest minor at time of writing[^6]. |

## References

[^1]: CloudNativePG project — originally built and sponsored by EDB. https://cloudnative-pg.io and repo README.
[^2]: CloudNativePG is a Cloud Native Computing Foundation Sandbox project. https://www.cncf.io/sandbox-projects/
[^3]: CloudNativePG architecture — the operator uses the Kubernetes API rather than an external distributed consensus store. https://cloudnative-pg.io/docs/
[^4]: "Why EDB Chose Immutable Application Containers." https://www.enterprisedb.com/blog/why-edb-chose-immutable-application-containers
[^5]: CNPG-I plugin interface and the Barman Cloud Plugin. https://github.com/cloudnative-pg/cnpg-i
[^6]: Release data from the GitHub API (repos/cloudnative-pg/cloudnative-pg/releases), retrieved 2026-07-15.

## Tags

kubernetes, postgresql, operator, high-availability, go, database, disaster-recovery, cncf, replication, backup, gitops
