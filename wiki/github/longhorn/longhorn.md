# longhorn/longhorn

> Distributed block storage for Kubernetes — a replicated CSI volume built entirely from containers and CRDs.

[GitHub repo](https://github.com/longhorn/longhorn) ·
[Official website](https://longhorn.io) ·
[License: Apache-2.0](https://github.com/longhorn/longhorn/blob/master/LICENSE)

## Overview

Longhorn is a distributed block storage system that runs as a set of Kubernetes workloads and exposes persistent volumes through a CSI driver. It was started at Rancher Labs (Sheng Yang) and open-sourced in 2017, donated to the CNCF as a Sandbox project in 2019, promoted to Incubating in 2021, and reached its 1.0 GA in mid-2020[^1]. Rancher — and therefore Longhorn — is now part of SUSE, and the security/support mailboxes still route to `suse.com`.

The design goal is deliberately unglamorous: give a bare-metal or edge Kubernetes cluster replicated, snapshot-capable persistent volumes without a separate storage appliance or a Ceph-scale operational commitment. Every volume gets its own dedicated storage controller (the "engine"), and each engine synchronously replicates writes to two or three replicas placed on different nodes' local disks. Because both the controller and the replicas are ordinary pods scheduled by Kubernetes, the whole system is self-hosting — there is no external control plane to run.

The defining tension is **simplicity versus raw performance**. Longhorn's original data path (the V1 engine) runs in userspace and presents volumes over an in-node iSCSI loopback, which is easy to reason about and portable but adds latency and CPU cost that make it a poor fit for high-IOPS databases. The newer V2 engine, built on SPDK, exists specifically to close that gap, at the price of hugepages, a supported NVMe/kernel setup, and less maturity. Most of the "should I use Longhorn?" decision comes down to which engine your workload needs. The repository itself (`longhorn/longhorn`) is primarily the umbrella: charts, manifests, docs, issues, and the release train; the actual code lives across a dozen sibling repos (`longhorn-manager`, `longhorn-engine`, `longhorn-instance-manager`, `longhorn-spdk-engine`, and others)[^2].

## Getting Started

Longhorn installs into an existing cluster with one manifest or a Helm chart. Nodes must have `open-iscsi` (V1) and a supported filesystem (`ext4`/`xfs`).

```bash
# Helm (recommended for production)
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --version 1.11.3

# or plain kubectl for a quick trial
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.11.3/deploy/longhorn.yaml
```

Once the `longhorn` StorageClass exists, provisioning is standard CSI:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

Run `longhornctl check preflight` (from `longhorn/cli`) before installing to catch missing `open-iscsi`, multipath conflicts, and kernel-module gaps.

## Architecture / How It Works

Longhorn models everything as Kubernetes Custom Resources reconciled by **longhorn-manager**, a DaemonSet controller. `Volume`, `Engine`, `Replica`, `Node`, `BackingImage`, `Snapshot`, and `Backup` are all CRDs; the manager watches them and drives the data-plane pods to match. There is no separate database — cluster etcd is the source of truth.

The data path, per volume:

1. **Engine (controller)** — one instance per attached volume, created and hosted by the **instance-manager** pod on the node where the workload runs. It receives all reads/writes and fans writes out to every replica synchronously; a write is acknowledged only once a quorum of replicas persists it.
2. **Replicas** — each replica is a sparse file tree on a node's local disk, stored as a chain of differencing images (the snapshot chain). Replicas live on distinct nodes so a single node loss never takes out a volume.
3. **iSCSI frontend (V1)** — the engine exposes the volume as an iSCSI target that the node's initiator mounts locally as a block device, which the CSI driver then formats and bind-mounts into the pod.

**Snapshots** are cheap because they are just new writable heads on the differencing chain; older snapshots become read-only layers. **Backups** walk that chain, detect changed 2 MiB blocks, and stream only the deltas to an NFSv4 or S3-compatible target via the `backupstore` library — so incremental backups move roughly the changed data, not the whole volume.

**ReadWriteMany** is not native. A RWX claim spins up a **share-manager** pod that mounts the underlying RWO Longhorn volume and re-exports it over NFS (Ganesha) to the consuming pods. This is convenient but means every RWX volume has an NFS gateway pod in its data path.

The **V2 engine** replaces the iSCSI/userspace path with SPDK: replicas become NVMe-oF targets and the engine runs a polled, kernel-bypass block stack. It reuses the same CRD model and manager, so from Kubernetes' perspective a V2 volume looks like a V1 volume with a different `dataEngine` field.

## Production Notes

**The V1 data path costs CPU and latency.** Because writes traverse userspace and an iSCSI loopback and are replicated synchronously, sustained random-write throughput and p99 latency are meaningfully worse than local disk. This is fine for most stateful apps (queues, small databases, CMS, CI caches) and painful for write-heavy OLTP. If IOPS matter, evaluate the V2/SPDK engine rather than tuning V1.

**Replica rebuilds are heavy.** When a replica is lost or a node drains, Longhorn rebuilds a full replica by streaming data across the network. On large volumes this saturates NIC bandwidth and competes with live I/O; schedule maintenance accordingly and watch the concurrent-rebuild limits. `dataLocality` (`best-effort`) keeps one replica co-located with the workload to cut read latency, but does not remove rebuild traffic.

**Node prerequisites bite silently.** Missing `open-iscsi`, an active `multipathd` grabbing Longhorn devices, or a filesystem Longhorn doesn't support are the most common install failures. Run the `longhornctl` preflight check; on multipath-enabled hosts you generally must blacklist Longhorn devices.

**RWX has a single-pod choke point.** Each ReadWriteMany volume's NFS export lives in one share-manager pod. If that pod is evicted, clients see an I/O stall until it reschedules and re-exports; failover is not instantaneous.

**Backups need discipline.** Recurring snapshots accumulate and consume disk if not pruned; configure retention. Backup targets must stay reachable, and restoring a large volume re-hydrates the full block set from object storage, which is bandwidth-bound. Test restores — a backup you have never restored is a hypothesis.

**Upgrades are staged, not atomic.** The control plane upgrades first (manager, CSI, instance-manager images), then volumes migrate to the new engine image, which for V1 can be done as a live, non-disruptive engine upgrade per volume. Read the per-release "Important Notes" page every time — Longhorn ships CRD schema changes and occasional one-way migrations between minor versions, and skipping intermediate minors is unsupported[^3].

**Capacity is thin-provisioned and can oversubscribe.** Volumes allocate lazily; a node whose actual disk fills while volumes think they have space will fail writes. Monitor real disk usage per node, not just the Longhorn-reported allocation.

## When to Use / When Not

**Use when:**
- You run stateful workloads on bare-metal, on-prem, or edge/K3s clusters with no cloud block-storage service to lean on.
- You want replicated volumes, snapshots, and incremental S3/NFS backups without operating Ceph.
- Your I/O profile is moderate (databases in the GB range, message queues, app data) rather than latency-critical OLTP.
- You value a GUI, CRD-driven operations, and a one-command install over maximum tunability.

**Avoid when:**
- You need top-tier random-write IOPS/latency and can't adopt the V2/SPDK engine.
- You already run Ceph at scale, or need object + file + block from one system (Rook/Ceph fits better).
- Your nodes can't meet the iSCSI/kernel/hugepages prerequisites (some managed or locked-down distros).
- You want RWX with no single-pod NFS gateway in the path.

## Alternatives

- rook/rook — Ceph-backed block, file, and object storage; far more capable and more operationally demanding. Use instead when you need multi-protocol storage at scale.
- openebs/openebs — sibling CNCF local/replicated storage project; its Mayastor engine is also SPDK-based. Use instead when you want a NVMe-focused replicated engine or lightweight local-PV options.
- piraeus-datastore/piraeus-operator — LINSTOR + DRBD replication on Kubernetes. Use instead when you want kernel-level synchronous replication and DRBD's maturity.
- ceph/ceph (direct, via Rook) — the reference distributed storage system. Use instead when Longhorn's scale ceiling or single-system-of-record needs are exceeded.
- Cloud CSI drivers (EBS, PD, Azure Disk) — use instead when you run in a cloud that already offers durable network block storage; Longhorn's value is highest where those don't exist.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-04 | Open-sourced by Rancher Labs[^1]. |
| Sandbox | 2019-10 | Accepted into the CNCF Sandbox. |
| 1.0 | 2020-06 | First GA release. |
| Incubating | 2021-11 | Promoted to CNCF Incubating[^1]. |
| 1.4 | 2023-01 | RWX improvements, backing-image and volume-clone maturation. |
| 1.5 | 2023-07 | V2 Data Engine (SPDK) introduced as preview[^2]. |
| 1.6 | 2024-02 | V2 engine, backup and volume enhancements. |
| 1.7 | 2024-09 | Continued V2 hardening, longhornctl CLI. |
| 1.9 | 2025 | Ongoing V2 progress toward production readiness. |
| 1.11 | 2026 | Latest stable branch (1.11.3)[^3]. |
| 1.12 | 2026 | Newest release branch (1.12.0). |

## References

[^1]: CNCF project profile, "Longhorn". https://www.cncf.io/projects/longhorn/
[^2]: Longhorn README, "Components" — source repositories and V1/V2 engine split. https://github.com/longhorn/longhorn
[^3]: Longhorn releases and per-version "Important Notes". https://github.com/longhorn/longhorn/releases

## Tags

kubernetes, storage, distributed-systems, block-storage, csi, cloud-native, cncf, high-availability, go, spdk
