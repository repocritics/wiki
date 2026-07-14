# openzfs/zfs

> Copy-on-write filesystem and volume manager with end-to-end checksums, snapshots, and pooled storage — for Linux and FreeBSD.

[GitHub repo](https://github.com/openzfs/zfs) ·
[Official docs](https://openzfs.github.io/openzfs-docs) ·
[License: CDDL-1.0](https://github.com/openzfs/zfs/blob/master/LICENSE)

## Overview

OpenZFS is a combined filesystem and volume manager originally developed at Sun Microsystems for Solaris (integrated ~2006) and now maintained by an independent community after Oracle closed the Solaris source[^1]. This repository is the Linux and FreeBSD implementation. It merges two historically separate ports — "ZFS on Linux" and FreeBSD's in-base ZFS — into a single shared codebase; that unification shipped as OpenZFS 2.0 in December 2020[^2]. FreeBSD's base system consumes this repo as of FreeBSD 13.0.

ZFS deliberately collapses the traditional storage stack. Instead of a partition table, a RAID layer (mdadm/hardware), a volume manager (LVM), and a filesystem, ZFS owns all of it: physical disks are grouped into *vdevs*, vdevs form a *pool* (`zpool`), and *datasets* (`zfs`) — filesystems and block volumes — allocate from the pool. Every block is checksummed and written copy-on-write, so the on-disk state is always consistent (no fsck) and silent corruption is detected and, with redundancy, self-healed on read or scrub.

The defining tension is licensing. ZFS is under the CDDL, which the FSF considers incompatible with the GPLv2 Linux kernel[^3]. The code therefore cannot be merged into mainline Linux and ships as an out-of-tree kernel module, built per-kernel via DKMS. This shapes everything downstream: distribution packaging is awkward, kernel upgrades can break the module, and the "just works" experience of an in-tree filesystem like ext4 or btrfs is not available. In exchange you get a feature set — cheap snapshots, send/receive replication, transparent compression, native encryption, RAID-Z — that no in-tree Linux filesystem fully matches.

## Getting Started

```bash
# Debian/Ubuntu (contrib repo; builds the DKMS module against your kernel)
sudo apt install zfsutils-linux zfs-dkms

# Fedora/RHEL: enable the openzfs repo, then
sudo dnf install zfs && sudo modprobe zfs
```

```bash
# Create a mirrored pool across two disks (use /dev/disk/by-id, not sdX)
sudo zpool create tank mirror \
  /dev/disk/by-id/ata-DISK1 /dev/disk/by-id/ata-DISK2

# A dataset with compression and hourly-snapshot-friendly layout
sudo zfs create -o compression=zstd -o atime=off tank/data

# Snapshot, then send it incrementally to another pool/host
sudo zfs snapshot tank/data@backup-1
sudo zfs send tank/data@backup-1 | sudo zfs recv backup/data
```

Always reference disks by `/dev/disk/by-id/*` — kernel `sdX` names are not stable across reboots, and a reordered pool can fail to import cleanly.

## Architecture / How It Works

The stack, bottom to top:

- **vdevs** — the redundancy unit. A vdev is a single disk, a mirror, or a RAID-Z group (RAID-Z1/2/3 tolerate 1/2/3 disk failures) or dRAID group. A pool stripes across its top-level vdevs; **there is no redundancy between vdevs**, so losing one non-redundant top-level vdev loses the whole pool.
- **Pool (SPA / Storage Pool Allocator)** — allocates blocks from vdevs, manages the `ashift` (sector size, fixed at creation), and holds the ZIL and ARC.
- **DMU (Data Management Unit)** — the transactional object layer. Groups writes into *transaction groups* (txg) flushed atomically every few seconds via copy-on-write, so a crash rolls back to the last consistent txg.
- **Datasets** — filesystems, zvols (block devices), snapshots, and clones, all sharing the pool's free space.

**Copy-on-write** means blocks are never overwritten in place: a modified block is written to a new location and parent pointers are rewritten up to the uberblock. Snapshots are therefore nearly free — they just pin the old block tree. **Checksums** are stored in the parent block pointer, not beside the data, giving true end-to-end integrity; a `scrub` walks the tree and repairs mismatches from redundant copies.

**Caching** is aggressive. The **ARC** (Adaptive Replacement Cache) lives in kernel RAM and is far more sophisticated than the page cache; **L2ARC** extends it onto SSD. Synchronous writes go through the **ZIL** (ZFS Intent Log), which can be placed on a fast **SLOG** device to lower latency. Feature evolution moved off monotonic pool version numbers (frozen at v28 after Oracle) to **feature flags**, so pools advertise individual capabilities rather than a single version[^4].

## Production Notes

**RAM is not optional.** The ARC will consume most free memory by default. On Linux, cap it with the `zfs_arc_max` module parameter — the default (half of RAM historically) surprises people on memory-constrained or shared hosts, and ARC accounting interacts poorly with tools that read `free`.

**Deduplication is a trap for most workloads.** Classic dedup keeps the entire dedup table (DDT) in RAM/ARC; underestimating it thrashes the pool and cannot be cheaply undone. Assume dedup is off unless you have measured the ratio. **Fast Dedup** in 2.3 (2025) reduces the memory penalty but does not make it free[^5].

**RAID-Z geometry is (mostly) permanent.** Until the **RAID-Z expansion** feature in 2.3, you could not add a disk to an existing RAID-Z vdev — you grew a pool by adding whole vdevs. Even with expansion, you cannot change RAID-Z level or shrink a vdev, and `ashift` is fixed at creation. Plan vdev layout up front.

**Keep pools under ~80% full.** As a copy-on-write allocator, ZFS fragments and slows dramatically near capacity; free space is also where snapshots and CoW churn live.

**The DKMS coupling bites on upgrades.** Because the module is out-of-tree, a kernel upgrade that outpaces OpenZFS support (or a failed DKMS rebuild) can leave a root-on-ZFS system unbootable. Pin kernels to versions OpenZFS supports and never remove the running kernel before the new module builds.

**Block cloning had a real incident.** The block-cloning feature introduced in 2.2.0 (2023) surfaced a data-corruption bug shortly after release; it was mitigated by disabling the feature and fixed in 2.2.2[^6]. A reminder that new on-disk features warrant caution on production data.

## When to Use / When Not

**Use when:**
- Data integrity matters more than raw simplicity — checksums plus scrub catch and repair bit rot that ext4/XFS silently pass through.
- You want snapshots, cheap clones, and `send`/`recv` replication as first-class storage primitives (backup servers, VM/container hosts, NAS).
- You run large multi-disk arrays and want the volume manager, RAID, and filesystem integrated and self-healing.

**Avoid when:**
- You need an in-tree, zero-friction Linux filesystem that survives arbitrary kernel upgrades (use ext4/XFS, or btrfs for CoW).
- The host is RAM-starved or you cannot budget for ARC.
- Your hardware is a single disk with no redundancy — ZFS detects corruption but cannot repair what it cannot reconstruct.
- You need to freely reshape RAID geometry over time on a tight disk budget.

## Alternatives

- btrfs — in-tree Linux CoW filesystem with snapshots; use when GPL/mainline integration and flexible reshaping outweigh ZFS's maturity and RAID-Z.
- LVM + mdadm + ext4/XFS — the conventional Linux stack; use when you want proven, in-kernel components and don't need checksums or snapshots-as-replication.
- ceph/ceph — distributed storage across many nodes; use when you need horizontal scale-out rather than a single-host pool.
- bcachefs — newer CoW filesystem with tiering; use when you want ZFS-like ideas in-tree and can accept much younger code.
- Oracle Solaris ZFS — the proprietary upstream fork; use only if you are locked into Solaris.

## History

| Version | Date | Notes |
|---------|------|-------|
| ZFS (Solaris) | 2006 | Originated at Sun; later open-sourced under CDDL[^1]. |
| ZoL 0.6.1 | 2013-03 | First "ZFS on Linux" release deemed production-ready. |
| 0.7.0 | 2017-07 | Performance work, `zpool` improvements. |
| 0.8.0 | 2019-05 | Native encryption, TRIM, sequential scrub/resilver[^7]. |
| 2.0.0 | 2020-12 | Linux + FreeBSD codebases unified; zstd compression[^2]. |
| 2.1.0 | 2021-07 | dRAID (distributed spare RAID-Z). |
| 2.2.0 | 2023-10 | Block cloning, BLAKE3 checksums, Linux container support[^6]. |
| 2.3.0 | 2025-01 | RAID-Z expansion, Fast Dedup, direct I/O[^5]. |

## References

[^1]: OpenZFS history and Solaris origin. https://openzfs.org/wiki/History
[^2]: OpenZFS 2.0.0 release — unified Linux/FreeBSD repository. https://github.com/openzfs/zfs/releases/tag/zfs-2.0.0
[^3]: FSF/SFC analysis of CDDL vs GPLv2 incompatibility. https://sfconservancy.org/blog/2016/feb/25/zfs-and-linux/
[^4]: OpenZFS feature flags (post-v28 versioning). https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Feature%20Flags.html
[^5]: OpenZFS 2.3.0 release notes — RAID-Z expansion, Fast Dedup. https://github.com/openzfs/zfs/releases/tag/zfs-2.3.0
[^6]: OpenZFS 2.2 block cloning corruption discussion. https://github.com/openzfs/zfs/issues/15526
[^7]: OpenZFS 0.8.0 release — native encryption. https://github.com/openzfs/zfs/releases/tag/zfs-0.8.0

## Tags

filesystem, storage, volume-manager, copy-on-write, zfs, linux-kernel, freebsd, raid, snapshots, data-integrity, c
