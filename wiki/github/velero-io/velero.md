# velero-io/velero

> Kubernetes-native backup, restore, and cross-cluster migration of cluster resources and their persistent volumes.

[GitHub repo](https://github.com/velero-io/velero) ·
[Official website](https://velero.io) ·
[License: Apache-2.0](https://github.com/velero-io/velero/blob/main/LICENSE)

## Overview

Velero backs up and restores Kubernetes cluster state — the API objects (Deployments, Services, ConfigMaps, CRDs, etc.) plus the data in their persistent volumes — to and from object storage. It began as Heptio Ark, was acquired with Heptio by VMware in 2018, renamed to Velero in 2019, and is now a CNCF project governed under the Linux Foundation[^1]. The repo moved from `vmware-tanzu/velero` to `velero-io/velero`; the old path redirects.

The mental model is deliberately narrow: Velero is not a snapshot tool bolted onto a cluster, it is a controller that reconciles `Backup` and `Restore` custom resources. A backup is a tarball of serialized Kubernetes objects written to a `BackupStorageLocation` (an S3/GCS/Azure bucket), optionally paired with volume snapshots taken either through the cloud provider's snapshot API or through file-level copy. Because the unit of work is "the set of objects matching a label/namespace selector," the same machinery serves three jobs: disaster recovery, migration between clusters, and cloning prod into staging.

The defining tension is snapshots vs. portability. Cloud-provider volume snapshots are fast and storage-efficient but are tied to a provider, region, and often a storage class — useless for migrating from EKS to on-prem. File-level backup (Restic, later Kopia) is portable across any storage backend but slow, and it reads through the filesystem rather than the block device. Every serious Velero deployment is a decision about which of these to use, and the answer is frequently "both, for different volumes."

## Getting Started

Install the CLI, then install the server components into the cluster with a provider plugin and a bucket:

```bash
brew install velero   # or download from the releases page

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent            # enables file-level (Kopia) volume backup
```

```bash
# Back up a namespace, including PV data via the node agent
velero backup create app-backup \
  --include-namespaces production \
  --default-volumes-to-fs-backup

# Restore it into a different (or the same) cluster
velero restore create --from-backup app-backup

# Schedule a daily backup with a 30-day TTL
velero schedule create daily --schedule="0 2 * * *" --ttl 720h0m0s
```

The `credentials-velero` file holds object-store credentials; the provider plugin is a separate image, versioned independently from Velero core.

## Architecture / How It Works

Velero is a set of controllers running in a single Deployment (`velero`) plus an optional `node-agent` DaemonSet. Everything is driven through CRDs:

- **`Backup` / `Restore`** — the top-level intent. A backup controller walks the API server for matching resources, serializes them, and streams a gzipped tarball to the `BackupStorageLocation`.
- **`BackupStorageLocation` / `VolumeSnapshotLocation`** — where object data and volume snapshots live. You can have several, e.g. a bucket per region.
- **`Schedule`** — a cron spec that periodically creates `Backup` objects.
- **`PodVolumeBackup` / `PodVolumeRestore`** — per-volume work items for the file-level path, executed by the node-agent on the node where the pod runs.
- **`DataUpload` / `DataDownload`** — work items for the CSI snapshot data mover, which promotes a provider snapshot into portable object storage.

Volume data has three distinct paths, and confusing them is the main source of operator error:

1. **Native cloud snapshots** — Velero calls the provider's snapshot API (via `VolumeSnapshotter` plugin). Fast, but the snapshot stays in the provider and is not portable.
2. **File-system backup (FSB)** — the node-agent reads the mounted volume and copies files into the object store. The uploader was Restic originally; **Kopia became the default uploader in v1.10** with the "unified repository" design, and the `restic` DaemonSet was renamed `node-agent`[^2].
3. **CSI snapshot data movement** — uses the Kubernetes CSI `VolumeSnapshot` API to take a snapshot, then a data mover copies the snapshot's contents to object storage so it survives cluster loss. This is the modern path for CSI-backed storage.

Restore is not a symmetric inverse of backup. Velero applies resources in a hardcoded priority order (namespaces, then CRDs, then the rest), strips cluster-assigned fields, and runs restore hooks. Ordering, `existingResourcePolicy`, and label-based inclusion/exclusion make restores the part of Velero most likely to behave unexpectedly.

Provider integrations (AWS, Azure, GCP, CSI, and third-party) are out-of-tree plugins communicating over a HashiCorp go-plugin gRPC interface, which is why plugin images are versioned and installed separately from core.

## Production Notes

- **File-system backup is slow and CPU/memory-hungry.** Kopia (and Restic before it) walks the filesystem and dedupes; on volumes with millions of small files, a single `PodVolumeBackup` can take hours and OOM the node-agent. Size the node-agent resource limits deliberately and prefer snapshot data movement for large volumes.
- **Node-agent backs up the pod's node.** FSB reads through the running pod's mounted volume, so the pod must be scheduled and the node-agent must run on that node (privileged, host path access). Volumes with no running pod are not covered by FSB.
- **Backups are crash-consistent, not application-consistent, by default.** Snapshotting a running database captures on-disk state, not a quiesced state. Use backup/restore hooks (`pre`/`post` exec) to flush and freeze — Velero will not do it for you.
- **Restore ordering and partial failures.** Large restores routinely finish with `PartiallyFailed`. Webhooks, admission controllers, mutating defaults, and immutable fields all interfere. Always test restores into a scratch cluster; a backup you have never restored is not a backup.
- **CRD and API-version drift across clusters.** Migration assumes the target cluster serves the same API versions. Backing up from an older cluster and restoring into a newer one where an API group was removed will silently drop those objects.
- **Credentials and IAM.** The object-store credentials in the `velero` secret are cluster-wide backup/restore power. IRSA / Workload Identity is preferred over static keys, but plugin support for it varies by version.
- **Kubernetes version support is n-2 for upgrades**; each release documents a tested-Kubernetes matrix, and combinations outside it are unverified rather than unsupported[^3].
- **`open_issues_count` is high** relative to activity — a large fraction are long-lived provider/edge-case reports, not a signal of abandonment; the project is actively maintained.

## When to Use / When Not

**Use when:**
- You need cluster-level disaster recovery covering both API objects and PV data.
- You are migrating or cloning workloads between clusters, clouds, or on-prem.
- You want backups stored in vendor-neutral object storage rather than locked in a provider snapshot service.
- You need scheduled, TTL'd, selector-scoped backups managed declaratively as CRDs.

**Avoid when:**
- You only need database backups — a database-native tool (pg_dump, WAL archiving, an operator's backup CR) gives application-consistent, point-in-time recovery Velero does not.
- You need continuous data protection / RPO measured in seconds — Velero is snapshot/schedule-based, not streaming replication.
- Your storage layer already provides array-level replication and you don't need Kubernetes object capture.
- You want a managed, GUI-first backup product with support SLAs out of the box (see the commercial layers built on Velero below).

## Alternatives

- vmware-tanzu/velero — same project (the old repo path; now a redirect to velero-io/velero).
- kanisterio/kanister — use instead when you need application-consistent, blueprint-driven backups (databases especially) rather than generic cluster capture.
- stashed/stash (KubeStash) — use when you want a broader Kubernetes backup platform with more built-in database integrations.
- longhorn/longhorn — use when you want backup/snapshot built into the storage layer itself rather than an orchestrator on top of it.
- restic/restic — use directly when you are backing up files/volumes without needing Kubernetes object capture or scheduling; Velero embeds this class of tool rather than replacing it.

## History

| Version | Date | Notes |
|---------|------|-------|
| Ark 0.x | 2017 | Released by Heptio as "Ark"; Heptio acquired by VMware in 2018. |
| 1.0 | 2019-05 | First GA under the Velero name[^1]. |
| 1.4 | 2020 | CSI snapshot support (beta), server-side restore improvements. |
| 1.9 | 2022 | Restic-based file backup maturing; groundwork for unified repository. |
| 1.10 | 2022-11 | Kopia becomes default uploader; `restic` DaemonSet renamed `node-agent`[^2]. |
| 1.12 | 2023 | CSI plugin merged toward core; snapshot data movement introduced. |
| 1.14–1.15 | 2024 | Data-mover maturation, resource modifiers, expanded CSI paths. |
| 1.18 | 2026 | Current line; tested against Kubernetes 1.33–1.35[^3]. |

## References

[^1]: Velero project history — Heptio Ark origin, VMware acquisition, and CNCF/Linux Foundation governance. https://velero.io/docs/main/how-velero-works/
[^2]: Velero 1.10 release notes — Kopia unified repository and `node-agent` rename. https://github.com/velero-io/velero/releases/tag/v1.10.0
[^3]: Velero README compatibility matrix and support process. https://velero.io/docs/latest/support-process/

## Tags

kubernetes, go, backup, disaster-recovery, restore, cloud-native, cncf, object-storage, persistent-volumes, migration, kopia, csi
