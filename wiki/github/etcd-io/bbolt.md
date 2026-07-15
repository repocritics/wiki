# etcd-io/bbolt

> A single-file, memory-mapped, B+tree key/value store for Go — the storage engine underneath etcd.

[GitHub repo](https://github.com/etcd-io/bbolt) ·
[Official website](https://go.etcd.io/bbolt) ·
[License: MIT](https://github.com/etcd-io/bbolt/blob/main/LICENSE)

## Overview

bbolt is an embedded key/value database for Go: a single file on disk, no server, no network, linked directly into your process. It is a fork of Ben Johnson's original Bolt (`boltdb/bolt`), itself modeled on Howard Chu's LMDB[^1]. When Bolt went into maintenance-only mode in 2017, the CoreOS/etcd team adopted the fork as `bbolt` to have an actively maintained target, and it now lives under the etcd-io organization[^2].

The data model is deliberately minimal: nested *buckets* (namespaces) containing byte-slice keys mapped to byte-slice values, kept in byte-sorted order. There are no indexes, no query language, no schema — just `Get`, `Put`, `Delete`, and cursor iteration. The defining architectural choice is a **single writer, many readers** concurrency model built on a copy-on-write B+tree and a memory-mapped file. Readers get a consistent snapshot with zero locking; writers serialize one at a time. This buys full ACID serializable transactions at the cost of write throughput.

bbolt's reach is largely a consequence of etcd: every etcd cluster stores its data in a bbolt file, so bbolt is transitively deployed in most Kubernetes control planes. Consul, and many Go projects that need a durable local store without operational overhead, use it directly. Its reputation is for correctness and stability rather than speed — it is the "boring, reliable" choice, and the maintenance mandate of the fork reflects that.

## Getting Started

```sh
go get go.etcd.io/bbolt@latest
```

```go
package main

import (
	"log"

	bolt "go.etcd.io/bbolt"
)

func main() {
	// A bbolt database is one file. It is created if absent.
	db, err := bolt.Open("my.db", 0600, &bolt.Options{Timeout: 1 * time.Second})
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Read-write transaction: exactly one runs at a time.
	err = db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucketIfNotExists([]byte("users"))
		if err != nil {
			return err
		}
		return b.Put([]byte("42"), []byte("Tom"))
	})
	if err != nil {
		log.Fatal(err)
	}

	// Read-only transaction: many run concurrently, lock-free.
	db.View(func(tx *bolt.Tx) error {
		v := tx.Bucket([]byte("users")).Get([]byte("42"))
		log.Printf("user = %s", v) // valid only inside the tx; copy() to keep it
		return nil
	})
}
```

A CLI is bundled for inspection, backup, and compaction: `go run go.etcd.io/bbolt/cmd/bbolt@latest`.

## Architecture / How It Works

The file is a sequence of fixed-size **pages** (typically the OS page size, 4KB). The entire file is `mmap`-ed into the process; reads are pointer dereferences into that mapping, so the OS page cache does the caching and bbolt keeps almost no in-process cache of its own.

- **B+tree.** Keys within a bucket live in a B+tree of branch and leaf pages. Iteration is byte-sorted and cheap because it walks contiguous, cache-friendly pages. Nested buckets are B+trees whose leaves point to other B+trees.
- **Copy-on-write / MVCC.** A write transaction never mutates live pages. It copies the pages it touches, builds a new tree, and commits by atomically writing a new **meta page** that points at the new root. There are two meta pages; commit alternates between them so a crash mid-write leaves the previous consistent version intact. This is what gives readers a stable snapshot without locks — an open read transaction simply keeps reading the meta it started with.
- **Freelist.** Pages orphaned by a committed write go on a freelist for reuse. bbolt has carried two freelist encodings — the original sorted array and a newer hashmap format that is faster for large freelists[^3].
- **Single writer.** A file lock plus an internal `sync.RWMutex`-style discipline enforces one writer at a time. `DB.Batch` opportunistically coalesces concurrent write closures into one fsync to amortize commit cost, at the price of the closure possibly running more than once (it must be idempotent).
- **Remap coupling.** When the file must grow, the writer re-`mmap`s it, which requires that no read transaction hold the old mapping. This is the source of the README's warning that mixing long-lived read transactions with writes in the same goroutine can deadlock.

Durability comes from an `fsync` on every commit by default. That fsync, not CPU, is the dominant cost of a write.

## Production Notes

The failure mode every bbolt operator eventually meets is **file bloat**, and it stems directly from the MVCC design:

- **The file never shrinks on its own.** Deleting keys returns pages to the freelist for reuse but does not return disk to the OS. A workload that grows then shrinks leaves a permanently large file. Reclaiming space requires compaction — `bbolt compact` offline, or online defragmentation (etcd's `etcdctl defrag` is exactly a bbolt compaction). Compaction rewrites the whole DB into a fresh file and needs room for both copies.
- **Long-running read transactions cause unbounded growth.** A read transaction pins the snapshot it opened, which prevents the writer from reusing any page freed since then. A single forgotten `View` held open under write load can make the file grow without limit — this is the classic etcd "database space exceeded" (`mvcc: database space exceeded`) incident. Keep read transactions short; never hold one across a request boundary or a blocking call.
- **Write throughput is bounded by one serialized writer plus one fsync per commit.** bbolt is excellent at reads and small/batched writes and poor at high-volume random writes. Batch writes with `DB.Batch`, or accept the latency floor set by your disk's fsync time. It is not a write-heavy ingest engine.
- **Memory usage is virtual, not resident** — the mmap means a 10GB DB maps 10GB of address space, but resident memory tracks the working set via the page cache. On 32-bit platforms the address space ceiling is a hard limit on DB size. Random-access workloads over larger-than-RAM datasets will page-fault to disk.
- **Values are borrowed, not owned.** A `[]byte` returned by `Get` (or seen in a cursor / `ForEach`) points into the mmap and is only valid until the transaction closes. Using it afterward is a use-after-unmap bug; `copy()` it out if it must outlive the tx.
- **`NoFreelistSync` / `NoSync`** are available for throughput but trade away crash guarantees — with `NoSync` a crash can corrupt the file. The `FreelistType` and `NoFreelistSync` options change how much freelist work happens on each commit; large freelists were a real commit-latency source that the hashmap freelist targets.
- **File format is stable and endian/page-size sensitive.** A DB file is portable across builds but not across differing OS page sizes; back up with `Tx.WriteTo` from a read transaction (a consistent hot backup) rather than copying the file out from under a live writer.

## When to Use / When Not

**Use when:**
- You want a durable, transactional local store with zero operational surface — one file, no daemon.
- Your workload is read-heavy with modest or batchable writes (config, metadata, indexes, embedded app state).
- You need ACID serializable transactions and consistent snapshots from a Go process.
- You value correctness and a frozen, well-tested format over raw write speed.

**Avoid when:**
- You have high-volume or write-heavy random-insert workloads — an LSM-tree store will sustain far higher write throughput.
- Your dataset is much larger than RAM with random access, or you need built-in compression.
- You need multiple processes writing the same database — bbolt takes an exclusive file lock (one process at a time).
- You need the file to reclaim disk automatically after deletes without a compaction step.

## Alternatives

- dgraph-io/badger — pure-Go LSM-tree with value-log separation (WiscKey); use it when writes are heavy or the dataset outgrows what a B+tree comfortably serves.
- cockroachdb/pebble — RocksDB-compatible LSM in Go (CockroachDB's engine); use it when you want LSM write performance with a mature, actively tuned Go implementation.
- syndtr/goleveldb — pure-Go LevelDB port; use it for an LSM store with a long track record when Badger/Pebble are heavier than needed.
- boltdb/bolt — the frozen original; do not start new projects on it, use bbolt instead (bbolt is the maintained continuation with the compatible API).
- LMDB (lmdb) — the C ancestor bbolt is modeled on; use it when you want the same mmap/B+tree design with C-level performance and can accept cgo or a non-Go dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| Bolt (boltdb/bolt) | 2014 | Ben Johnson's original pure-Go Bolt, modeled on LMDB[^1]. |
| Bolt maintenance mode | 2017 | Original author freezes Bolt; API and file format declared stable[^1]. |
| bbolt fork | 2017 | CoreOS/etcd forks Bolt as bbolt for active maintenance; import path `go.etcd.io/bbolt`[^2]. |
| v1.3.x | 2017 onward | Long-lived stable series; bug fixes, cursor-delete correctness, tooling[^4]. |
| v1.4.0 | 2025 | Major release: structured logging, freelist/format improvements, cleanup of legacy options[^3][^4]. |

## References

[^1]: bbolt README — provenance as a fork of Ben Johnson's Bolt, itself inspired by Howard Chu's LMDB. https://github.com/etcd-io/bbolt/blob/main/README.md
[^2]: etcd-io/bbolt repository, under the etcd-io organization; used as etcd's storage backend. https://github.com/etcd-io/bbolt
[^3]: bbolt freelist encodings (array vs hashmap) and `FreelistType` / `NoFreelistSync` options — package documentation. https://pkg.go.dev/go.etcd.io/bbolt
[^4]: bbolt releases. https://github.com/etcd-io/bbolt/releases

## Tags

go, embedded-database, key-value-store, b-plus-tree, mmap, acid, transactions, storage-engine, etcd, database
