# dgraph-io/badger

> Embeddable, persistent key-value store in pure Go, built on the WiscKey design that keeps values out of the LSM tree.

[GitHub repo](https://github.com/dgraph-io/badger) ·
[Official website](https://dgraph-io.github.io/badger/) ·
[License: Apache-2.0](https://github.com/dgraph-io/badger/blob/main/LICENSE)

## Overview

BadgerDB is an in-process key-value database written in pure Go (no cgo), created by Dgraph Labs (now Hypermode) as the storage engine underneath the Dgraph graph database[^1]. It exists because the Go ecosystem lacked a fast embedded LSM store that did not require binding to a C++ engine like RocksDB — cgo complicates cross-compilation, breaks the Go race detector at the boundary, and adds a build-toolchain dependency. Badger is that store: `go get` it and you have a transactional, SSD-oriented KV engine linked directly into your binary.

Its defining design choice is **key-value separation**, taken from the WiscKey paper[^2]. A conventional LSM tree (LevelDB, RocksDB) stores keys and values together in its sorted string tables, so every compaction rewrites the values too — large values cause large write amplification. Badger instead writes values to a separate append-only **value log** and stores only a pointer (file id + offset) alongside the key in the LSM tree. The tree stays small enough to keep largely in memory, point lookups become an LSM read plus one value-log seek, and compaction moves pointers rather than payloads. This wins decisively for workloads with values bigger than a few hundred bytes; for very small values the extra indirection and value-log bookkeeping is closer to a wash.

The tradeoff Badger imposes in return is **value-log garbage collection**. Because values are appended and never updated in place, stale versions accumulate in the log and must be reclaimed by the application calling GC explicitly. This is the single most common source of production surprise (see Production Notes) and the price of the WiscKey design.

## Getting Started

```sh
go get github.com/dgraph-io/badger/v4
```

The import path is versioned: v4 is the current major line and uses the `/v4` module suffix[^3].

```go
package main

import (
	"log"

	badger "github.com/dgraph-io/badger/v4"
)

func main() {
	db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Read-write transaction.
	err = db.Update(func(txn *badger.Txn) error {
		return txn.Set([]byte("answer"), []byte("42"))
	})
	if err != nil {
		log.Fatal(err)
	}

	// Read-only transaction.
	err = db.View(func(txn *badger.Txn) error {
		item, err := txn.Get([]byte("answer"))
		if err != nil {
			return err
		}
		return item.Value(func(val []byte) error {
			log.Printf("answer = %s", val)
			return nil
		})
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

`db.Update` / `db.View` wrap the transaction lifecycle and retry-free commit; for manual control use `db.NewTransaction(update bool)`. Values are exposed through a callback (`item.Value`) rather than returned directly because the byte slice is only valid for the duration of the transaction — a frequent correctness footgun for newcomers who retain the slice.

## Architecture / How It Works

Badger is an **LSM tree plus value log**:

- **Memtable / WAL.** Writes land in an in-memory skiplist memtable and a write-ahead log. When the memtable fills it is flushed to an immutable Level-0 SSTable on disk.
- **LSM tree.** SSTables are organized in levels; background compaction merges them downward, discarding overwritten and expired keys. Because the tree holds only keys plus small value pointers, it is compact and its index/bloom-filter structures can be memory-resident or memory-mapped.
- **Value log (vlog).** Values larger than `ValueThreshold` are written to append-only vlog files; smaller values are inlined into the LSM tree to avoid the extra seek. The threshold is tunable.
- **MVCC + transactions.** Every key carries a monotonically increasing version (timestamp) from a global oracle. Transactions use **serializable snapshot isolation**: reads see a consistent snapshot, and on commit the oracle detects read-write conflicts and aborts (`ErrConflict`) rather than corrupting. A Jepsen-style bank test runs nightly to exercise these guarantees[^1].
- **3D access.** Keys are addressable by (key, value, version), so the iterator can walk historical versions; `Options` controls how many versions per key are retained.

Two operating modes exist. The default mode manages versions internally. **Managed mode** (`OpenManaged`) hands version/timestamp control to the caller — this is what Dgraph uses to coordinate Badger with its own distributed transaction layer, and it changes the API contract enough that most applications should stay on the default.

On-disk files are memory-mapped, and Badger uses OS-specific code for directory locking and `fsync`. POSIX systems (Linux, macOS, BSD) get full durability; Windows uses a different locking path, and Plan9 / WASM builds have no file locking at all[^4].

## Production Notes

- **You must run value-log GC yourself.** Nothing reclaims stale values automatically. The documented pattern is to call `db.RunValueLogGC(discardRatio)` periodically (commonly on a ticker), looping until it returns `ErrNoRewrite`. Skip this and disk usage grows without bound even as the logical dataset stays flat — the most-reported Badger operational issue. GC is opportunistic: it only rewrites a vlog file when it estimates enough dead data, so a single call reclaims at most one file.
- **Memory footprint is real.** Table indexes, bloom filters, and the block cache are held in memory; a large dataset with default options can consume gigabytes of RAM. On memory-constrained hosts tune `Options.WithNumMemtables`, `WithBlockCacheSize`, `WithIndexCacheSize`, and (historically) `WithTableLoadingMode` / `WithValueLogLoadingMode` to trade RAM for I/O. Loading everything via mmap can also drive up resident memory unexpectedly.
- **Crash durability depends on SyncWrites.** With `SyncWrites` disabled (higher throughput), recent writes since the last sync can be lost on power failure; with it enabled every commit `fsync`s and throughput drops. Choose deliberately.
- **Not for concurrent processes.** Badger takes an exclusive directory lock — exactly one process opens a directory at a time. It is an embedded library, not a server; sharing data across processes means putting your own service in front of it.
- **Discard stats and version bloat.** Keeping many versions per key (or long TTLs with slow GC) inflates both the LSM tree and the vlog. `db.Flatten()` and explicit compaction help, but the durable fix is bounding versions via options.
- **Major-version upgrades are not drop-in.** The on-disk format has changed across v1→v2→v3→v4, and the import path changes with each major (`/v2`, `/v3`, `/v4`). Migrations generally require a backup/restore (`badger backup` / `badger restore`) rather than an in-place open. Pin the major version and test the upgrade against a copy of real data.
- **Go version coupling.** The maintainers intentionally hold the `go.mod` minimum low to avoid forcing downstream Go upgrades, so it builds against a wide range of toolchains.

## When to Use / When Not

**Use when:**
- You need an embedded transactional KV store in a Go binary without cgo.
- Your values are medium-to-large and write throughput matters — the WiscKey separation pays off.
- You want ACID transactions with snapshot isolation inside a single process (metadata stores, blockchain nodes, tracing/profiling backends, caches).
- You are on SSD/NVMe, which the design assumes.

**Avoid when:**
- You want zero background maintenance — bbolt's B+tree has no value-log GC to schedule.
- Your workload is tiny keys and tiny values, read-mostly — a B+tree store is simpler and competitive.
- You need multiple processes or machines to share the data — Badger is single-process; use a networked database or wrap it in a service (as Dgraph does).
- You cannot budget the RAM its in-memory indexes and caches expect.

## Alternatives

- etcd-io/bbolt — pure-Go B+tree, single-writer, memory-mapped; use it when reads dominate and you want zero background GC and a dead-simple durability model.
- cockroachdb/pebble — pure-Go LSM (RocksDB-compatible semantics) powering CockroachDB; use it when you want RocksDB-style tuning and a large-scale-proven engine without cgo.
- facebook/rocksdb — the mature C++ LSM engine; use it (via cgo bindings) when you need its ecosystem, tooling, and tuning knobs and can accept the build complexity.
- syndtr/goleveldb — pure-Go LevelDB port; use it when you want a lighter, simpler LSM without value-log separation or Badger's memory appetite.
- dgraph-io/ristretto — not a store but a concurrent cache from the same authors; use it when you only need an in-memory cache rather than a persistent database.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2017 | Initial release; WiscKey-based key-value separation[^1][^2]. |
| v2.0 | 2020 | On-disk format changes; encryption-at-rest and related features moved into the open-source engine. |
| v3.0 | 2021 | Reworked memtable/WAL and compaction; `/v3` module path. |
| v4.0 | 2023 | Current major line; `/v4` module path, cache and options refinements. |

## References

[^1]: BadgerDB README — project status, design goals, and nightly Jepsen-style bank test. https://github.com/dgraph-io/badger
[^2]: Lu et al., "WiscKey: Separating Keys from Values in SSD-conscious Storage," USENIX FAST 2016 — the paper Badger's design is based on. https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf
[^3]: Go package reference for `github.com/dgraph-io/badger/v4`. https://pkg.go.dev/github.com/dgraph-io/badger/v4
[^4]: BadgerDB README — platform compatibility table (`dir_*.go` per-OS locking and fsync behavior). https://github.com/dgraph-io/badger#platform-compatibility

## Tags

go, golang, key-value-store, embedded-database, lsm-tree, wisckey, database, storage-engine, acid-transactions, ssd, nosql
