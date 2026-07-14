# rook/rook

> A Kubernetes operator that deploys and manages Ceph — turning a notoriously complex distributed storage system into CRDs.

[GitHub repo](https://github.com/rook/rook) ·
[Official website](https://rook.io) ·
[License: Apache-2.0](https://github.com/rook/rook/blob/master/LICENSE)

## Overview

Rook is a cloud-native storage orchestrator for Kubernetes. In practice, "orchestrator" means it is a Kubernetes operator that runs [Ceph](https://ceph.com/) — a distributed storage system providing block, file, and object storage — as a set of pods inside your cluster, driven entirely by custom resources[^1]. You declare a `CephCluster` and related CRDs; the operator reconciles them into running Ceph daemons (monitors, managers, OSDs, metadata servers, object gateways) and wires up CSI drivers so PersistentVolumeClaims are backed by Ceph.

Rook was originally a multi-backend framework — early versions shipped operators for EdgeFS, Cassandra, NFS, CockroachDB, YugabyteDB, and MinIO alongside Ceph. Those were progressively deprecated and removed; by around 2021 Rook was Ceph-only, and the project now describes itself squarely as Ceph-on-Kubernetes[^2]. This history matters: much of the abstraction machinery (the "storage provider framework") was designed for generality that no longer exists, and the docs, CRDs, and operator are now purpose-built for Ceph.

The defining tension is that **Rook automates Ceph's deployment but does not hide Ceph's complexity.** It removes the toil of placing daemons, formatting disks, and rolling upgrades, but a healthy Rook cluster still requires understanding Ceph concepts: CRUSH maps, placement groups, mon quorum, OSD failure domains, and pool replication. Rook is a Kubernetes-native control plane over Ceph, not a way to avoid learning Ceph.

Rook is a CNCF graduated project — the first storage project to reach that tier, graduating in October 2020[^3].

## Getting Started

Rook is installed as a Helm chart or raw manifests. The operator runs first; the cluster CR comes second.

```bash
# Install the operator (Helm)
helm repo add rook-release https://charts.rook.io/release
helm install --create-namespace --namespace rook-ceph \
  rook-ceph rook-release/rook-ceph

# Install a CephCluster + default StorageClasses
helm install --namespace rook-ceph \
  rook-ceph-cluster rook-release/rook-ceph-cluster
```

A minimal cluster CR consuming all raw devices on every node:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.4
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3            # odd number for quorum
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: true # consumes every unused raw block device
```

Once healthy, a `CephBlockPool` + `StorageClass` (installed by the cluster chart) lets any PVC provision RBD block storage. The `rook-ceph-tools` pod gives you a shell with the `ceph` CLI for direct inspection.

## Architecture / How It Works

Rook is a single operator process that watches Ceph CRDs (`ceph.rook.io/v1`) and reconciles them into Kubernetes workloads. The Ceph daemons themselves are ordinary pods:

- **MON (monitors)** — maintain cluster maps and quorum. Rook runs an odd count (typically 3) as separate deployments and manages failover by replacing lost mons.
- **MGR (managers)** — host the Ceph dashboard, Prometheus exporter, and orchestration modules.
- **OSD (object storage daemons)** — one per backing disk; the actual data plane. Rook discovers devices, provisions OSDs (on raw devices, partitions, or PVCs), and manages them via `ceph-volume`.
- **MDS** — metadata servers for CephFS (file). **RGW** — RADOS gateways for object storage (S3/Swift).

Consumption happens through **Ceph-CSI**, a separate project Rook deploys and configures. The RBD CSI driver backs block PVCs; the CephFS CSI driver backs shared filesystem PVCs; object storage is exposed via `CephObjectStore` + `ObjectBucketClaim`. Rook generates the CSI secrets and StorageClasses so this is mostly invisible to app teams.

The reconciliation model is level-triggered: the operator continuously drives observed state toward the CR spec. Upgrades are expressed by changing image tags in the CR — Rook orchestrates a rolling daemon restart in the correct order (mons, then mgrs, then OSDs, etc.) and gates each step on Ceph health.

The critical coupling to understand is the **two-layer version matrix**: Rook (the operator) and Ceph (the payload) version independently. A given Rook release supports a specific range of Ceph versions, and each must be upgraded on its own supported path. Mismatches are a common source of stuck reconciliation.

## Production Notes

**Ceph knowledge is non-optional.** When a cluster goes `HEALTH_WARN` — an OSD down, PGs undersized, a full ratio hit — Rook surfaces it but does not fix it for you. Operators end up in the `rook-ceph-tools` pod running `ceph status`, `ceph osd tree`, and `ceph pg` commands. Teams that adopt Rook expecting to never touch Ceph are the ones who get burned.

**Sizing and failure domains.** Default replicated pools want 3 replicas across 3 distinct nodes. On a 3-node cluster, losing one node means no room to re-replicate; recovery blocks until the node returns. Production clusters run more nodes and set the CRUSH failure domain deliberately (host vs. rack vs. zone). Do not run production storage on 2 nodes.

**`dataDirHostPath` is stateful and dangerous.** Mon and OSD metadata live under this host path. Deleting and recreating a `CephCluster` without wiping `dataDirHostPath` and the OSD disks leaves stale state that breaks the new cluster. Full teardown requires the documented cleanup procedure (`cleanupPolicy` + zapping devices); skipping it is a frequent support issue.

**OSD-on-PVC vs. OSD-on-host.** Two provisioning models with different operational characteristics. Host-based OSDs (raw devices on nodes) are simplest for bare metal. PVC-based OSDs (backing Ceph with cloud block volumes) enable dynamic scaling but add a storage layer under your storage layer and can hurt performance and cost.

**Upgrades are multi-step and order-sensitive.** You upgrade Rook first (within its supported version skew), then Ceph, following the release notes' matrix. Jumping multiple minor versions at once is unsupported. Always check `ceph status` is `HEALTH_OK` before starting; an upgrade that begins on an unhealthy cluster can stall mid-rollout.

**Networking.** By default Ceph traffic uses the pod network. High-throughput clusters often want host networking or a multus-based dedicated storage network; changing this later is disruptive. The mon endpoints are also sensitive — losing mon quorum takes the whole cluster read/write offline.

**Resource requests.** OSDs are memory-hungry (BlueStore cache). Under-provisioning memory requests leads to OOM-killed OSDs and flapping. Set explicit requests/limits rather than trusting defaults.

## When to Use / When Not

**Use when:**
- You run Kubernetes on bare metal or on-prem and need block/file/object storage without an external SAN.
- You already run Ceph, or have Ceph expertise, and want a Kubernetes-native control plane for it.
- You need all three storage types (block via RBD, shared file via CephFS, S3-compatible object via RGW) from one system.
- You want dynamic PV provisioning independent of a specific cloud provider.

**Avoid when:**
- You're on a managed cloud where EBS/PD/Azure Disk CSI already gives you cheap, reliable block storage — Rook adds a distributed system you must operate.
- Your cluster is tiny (1–2 nodes) or short-lived; Ceph's replication and quorum assumptions don't fit.
- Nobody on the team wants to learn Ceph. Rook lowers the deployment barrier, not the operational one.
- You only need object storage and a hosted S3 (or MinIO) is acceptable.

## Alternatives

- longhorn/longhorn — simpler distributed block storage for Kubernetes; easier to operate, block-only, no CephFS/object.
- openebs/openebs — container-attached storage with multiple engines (Mayastor/NVMe-oF); use when you want per-workload local/replicated volumes over a single Ceph cluster.
- ceph/ceph — run Ceph directly (via cephadm) when Ceph isn't primarily serving Kubernetes or you want the cluster decoupled from the k8s lifecycle.
- minio/minio — use when you only need S3-compatible object storage and don't want a full Ceph deployment.
- piraeus-datastore/piraeus-operator — LINSTOR/DRBD-based replicated block storage; an alternative when you want DRBD semantics instead of Ceph.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-07 | Repo created; early multi-backend storage framework[^1]. |
| — | 2018-01 | Accepted into the CNCF as a sandbox (then inception) project[^3]. |
| 1.0 | 2019-05 | Ceph operator declared stable; storage-provider framework matured. |
| — | 2020-10 | Graduated within the CNCF — first storage project to do so[^3]. |
| ~1.8 | 2021–2022 | Non-Ceph providers (Cassandra, NFS, EdgeFS, etc.) removed; Rook becomes Ceph-only[^2]. |
| 1.x | ongoing | Ceph-focused releases tracking upstream Ceph (Quincy, Reef, Squid) with a per-release version matrix. |

## References

[^1]: Rook README — "Storage Orchestration for Kubernetes." https://github.com/rook/rook
[^2]: Rook documentation, storage architecture (Ceph-only). https://rook.io/docs/rook/latest-release/Getting-Started/storage-architecture/
[^3]: CNCF, "Cloud Native Computing Foundation Announces Rook Graduation" — 2020-10-07. https://www.cncf.io/announcements/2020/10/07/cloud-native-computing-foundation-announces-rook-graduation/

## Tags

go, kubernetes, ceph, storage, cloud-native, cncf, operator, distributed-storage, csi, persistent-volumes
