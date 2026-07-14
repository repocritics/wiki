# minio/minio

> High-performance, S3-compatible object store in a single Go binary — now archived, with the community edition frozen to source-only distribution.

[GitHub repo](https://github.com/minio/minio) ·
[Official website](https://min.io) ·
[License: AGPL-3.0](https://github.com/minio/minio/blob/master/LICENSE)

## Overview

MinIO is an object storage server that speaks the Amazon S3 API and runs as a single, dependency-free Go binary. It became the default self-hosted "S3 on your own hardware" for a decade: `minio server /data`, point any S3 SDK at it, done. It is written in Go, uses Reed-Solomon erasure coding for durability instead of an external database or replication layer, and scales from a laptop to petabyte clusters with the same binary.

The defining tension is governance, not technology. MinIO Inc. relicensed the project from Apache-2.0 to GNU AGPLv3 in April 2021[^1], which makes embedding or offering it as a managed service a legal decision, not just a technical one. In 2025 the company stripped most administrative features out of the community web console, steering operators toward the commercial AIStor product[^2]. As of 2026 the `minio/minio` repository is **archived** and the community edition is distributed as **source only** — no maintained pre-compiled binaries, no ongoing feature work[^3]. The 61k-star history reflects how ubiquitous it became; the archived status reflects that the vendor has moved active development behind a commercial line.

Read this page as a catalog entry for infrastructure that is enormously deployed but no longer maintained upstream. New greenfield deployments should weigh the alternatives below.

## Getting Started

The community edition no longer ships binaries; you build from source (Go 1.24+) or a container[^3].

```sh
# Install from source
go install github.com/minio/minio@latest

# Run a standalone server against an empty directory
minio server /data --console-address :9001
# default root credentials: minioadmin:minioadmin
```

```sh
# Talk to it with the mc client
mc alias set local http://localhost:9000 minioadmin minioadmin
mc mb local/mybucket
mc cp ./file.txt local/mybucket/
mc ls local/mybucket/
```

Any S3 SDK works unchanged — set the endpoint to `http://localhost:9000`, use path-style addressing, and the `minioadmin` keys as access/secret.

## Architecture / How It Works

MinIO deliberately has no metadata database, no cache tier, and no external coordinator. Everything lives on the drives.

- **Erasure coding.** Objects are split into data and parity shards (Reed-Solomon) and striped across drives in an *erasure set* — typically up to 16 drives. Default parity tolerates the loss of up to half the drives in a set while keeping data readable. Per-object metadata is stored inline alongside the shards (`xl.meta`), so small objects need no extra lookups. Bitrot is detected on read via per-shard checksums and healed from parity.
- **No erasure coding in the trivial case.** A single-node single-drive deployment (`minio server /data` on one path) has *no* redundancy — it is a plain filesystem-backed store. Erasure coding requires multiple drives (minimum four).
- **Server pools.** A distributed deployment is one or more *pools*, each a fixed grid of nodes × drives that MinIO divides into erasure sets at startup. Consistency is strict read-after-write. There is no rebalancing across pools by default: adding capacity means adding a whole new pool, and existing objects stay where they were written unless you explicitly run `mc admin rebalance`.
- **S3 surface.** Versioning, object locking / WORM retention, lifecycle transitions, bucket and site replication, presigned URLs, and server-side encryption (SSE-S3/SSE-KMS via the KES key server, or SSE-C) are all implemented. IAM is internal S3-style JSON policies, optionally federated to LDAP or OIDC.
- **The console.** MinIO embeds a web console for browsing buckets and, historically, administration. The administrative half of that console was removed from the community build in 2025[^2]; object browsing remains.

The design decision that made MinIO easy — geometry (nodes × drives) is fixed at deploy time and encoded in the launch command — is also the decision that makes it rigid in production.

## Production Notes

- **The repo is archived.** No new binaries, security patches, or fixes are guaranteed for the community edition[^3]. Treat any current deployment as frozen and plan a migration or a pinned, self-maintained build. Community forks that restore the removed console features exist, but none carries the original team's backing.
- **You cannot resize an erasure set.** The drive/node count of a pool is immutable. To grow, you add another pool with its own matching geometry; you cannot add a single drive to an existing set. Shrinking requires draining and redeploying.
- **Heterogeneous drives waste space.** Erasure sets treat all drives as the smallest member; mixing 4 TB and 12 TB drives strands the difference. Keep drive sizes uniform within a pool.
- **No automatic rebalance.** After adding a pool, new writes are spread by free-space weighting, but existing data does not move until you run `mc admin rebalance start`. Clusters expanded naively end up with one hot pool.
- **AGPLv3 is a real constraint.** Offering MinIO as a service, embedding it in a shipped product, or exposing modified versions over a network triggers source-disclosure obligations[^1]. Legal review is not optional for commercial use; this is the single most common reason teams choose an alternative.
- **Gateway mode is gone.** The old NAS/S3/Azure/GCS gateway modes were deprecated and removed in 2022[^4]; MinIO is a storage server, not a translation proxy, on current code.
- **Defaults are unsafe.** `minioadmin:minioadmin` must be rotated immediately, and the console/API ports fronted by TLS. The server will run wide open otherwise.
- **Upgrades were historically in-place and frequent.** MinIO used date-stamped `RELEASE.YYYY-MM-DDTHH-MM-SSZ` tags, not semver; rolling restarts were the norm and occasional releases changed on-disk or IAM behavior. With the repo archived, the upgrade treadmill has simply stopped.

## When to Use / When Not

**Use when:**
- You need an S3 API on your own hardware today and can pin to a specific, self-maintained build.
- Your workload is large objects (analytics, ML datasets, backups) where erasure coding and throughput shine.
- You are AGPLv3-compatible: internal use, or a fully open-source stack.

**Avoid when:**
- You are starting fresh in 2026 and want an upstream that is actively maintained — the community repo is archived.
- You need to embed object storage in a proprietary product or a hosted service without a commercial license.
- Your data is millions of tiny files, or you need block/file protocols alongside object — other systems fit better.
- You need to grow capacity one drive at a time; MinIO's pool model does not allow it.

## Alternatives

- ceph/ceph — full distributed storage (object via RGW, plus block and file); far heavier to operate but battle-tested and vendor-neutral. Use when you need more than object storage or want to avoid single-vendor risk.
- seaweedfs/seaweedfs — Go object store optimized for huge numbers of small files with an S3 gateway. Use when your workload is many small objects rather than large ones.
- apache/ozone — Hadoop-ecosystem object store with S3 compatibility. Use when you live in the HDFS/analytics world and want native integration.
- deuxfleurs-org/garage — lightweight, geo-distributed S3-compatible store designed for small self-hosted multi-site setups. Use when you want simple, low-footprint, multi-region durability.
- versity/versitygw — actively maintained S3 gateway that fronts POSIX filesystems and other backends. Use when you want an S3 face over existing storage rather than a new storage engine.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repository created | 2015-01 | Early MinIO server development on GitHub. |
| Erasure-coded distributed mode | 2016–2017 | Reed-Solomon striping across drives becomes the core durability model. |
| Relicensed to AGPLv3 | 2021-04 | Moved from Apache-2.0 to GNU AGPLv3[^1]. |
| Gateway modes removed | 2022 | NAS/S3/Azure/GCS translation gateways deprecated and dropped[^4]. |
| Community console stripped | 2025 | Administrative features removed from the open-source console; pushed toward AIStor[^2]. |
| Source-only + archived | 2026 | No maintained community binaries; `minio/minio` repository archived[^3]. |

MinIO never used semantic versioning; releases were date-stamped `RELEASE.*` tags, which is why the table above tracks milestones rather than version numbers.

## References

[^1]: MinIO, "MinIO Server Now Available Under AGPLv3" — announcement of the Apache-2.0 → GNU AGPLv3 relicense, April 2021. https://blog.min.io/from-open-source-to-agplv3/
[^2]: Community discussion of administrative features removed from the MinIO Object Browser / console in 2025. https://github.com/minio/object-browser
[^3]: `minio/minio` README and repository status — repository archived, community edition distributed as source only, no maintained pre-compiled binaries. https://github.com/minio/minio
[^4]: MinIO documentation and changelog — deprecation and removal of gateway modes (NAS/S3/Azure/GCS), 2022. https://min.io/docs/minio/linux/index.html

## Tags

go, object-storage, s3-compatible, erasure-coding, distributed-storage, self-hosted, cloud-native, kubernetes, agpl, archived, storage-server
