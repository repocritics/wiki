# cockroachdb/pebble

> A RocksDB-inspired LSM key-value store in pure Go — CockroachDB's storage
> engine first, a general-purpose library second.

[GitHub repo](https://github.com/cockroachdb/pebble) ·
[API docs](https://pkg.go.dev/github.com/cockroachdb/pebble/v2) ·
[Nightly benchmarks](https://cockroachdb.github.io/pebble/) ·
License: BSD-3-Clause

## Overview

Pebble is an embedded LSM-tree key-value store written in Go, created by
Cockroach Labs to replace RocksDB inside CockroachDB[^1]. The motivations were
concrete: eliminate the cgo boundary (every RocksDB call from Go crossed an
FFI wall) and own the engine so its roadmap could track CockroachDB's needs
exactly. It shipped as an optional engine in CockroachDB 20.1 (May 2020),
became the default in 20.2 (November 2020)[^2], and has run at scale ever
since. Outside CockroachDB, go-ethereum adopted it as a database backend[^3].

The defining tension is stated in the README itself: Pebble "specifically
targets the use case and feature set needed by CockroachDB." It is arguably
the most rigorously engineered LSM in the Go ecosystem — nightly benchmarks,
metamorphic testing, years of production hardening — but its feature set is
curated by one consumer: no transactions, no column families, no backups, no
universal or FIFO compaction. The ~6k stars understate its reach; it is
infrastructure most people run indirectly.

Pebble inherited RocksDB's file formats; the `v1` series stayed
forward-compatible with RocksDB 6.2.1 databases. `v2` (January 2025) cut that
cord — it cannot open RocksDB-generated stores at all.

## Getting Started

```bash
go get github.com/cockroachdb/pebble/v2
```

```go
package main

import (
    "fmt"
    "log"

    "github.com/cockroachdb/pebble/v2"
)

func main() {
    db, err := pebble.Open("demo", &pebble.Options{})
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if err := db.Set([]byte("hello"), []byte("world"), pebble.Sync); err != nil {
        log.Fatal(err)
    }
    value, closer, err := db.Get([]byte("hello"))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("hello %s\n", value) // value valid only until closer.Close()
    _ = closer.Close()
}
```

## Architecture / How It Works

Pebble is a classic write-ahead-log + memtable + leveled-SSTable LSM, with
deliberate departures from RocksDB[^4]:

- **Memtable** — an arena-backed lock-free skiplist with backward links,
  making reverse iteration cheap (a known RocksDB weak spot).
- **Commit pipeline** — a redesigned group-commit path with better write
  concurrency; one of the original motivations for the rewrite.
- **L0 sublevels + flush splitting** — L0 is organized into sublevels so
  multiple compactions can drain it concurrently under heavy write load.
- **Copy-on-write B-tree of file metadata** — installing a compaction/flush
  result in an LSM with hundreds of thousands of sstables is cheap because
  file metadata is a persistent data structure, not a rewritten array.
- **Block-property collectors and filters** — user-defined per-block/table
  properties (CockroachDB uses MVCC timestamps) let iterators skip whole
  tables, index blocks, and data blocks.
- **Range keys** — first-class ranged KV pairs interleaved during iteration
  (CockroachDB's MVCC range tombstones sit on this).
- **Delete-only compactions** and **virtual sstables** — whole-file drops
  under range deletions; logical slicing of sstables for ingest/excise.
- **Columnar blocks** — `v2` introduced a columnar sstable block layout
  (`FormatColumnarBlocks`) replacing the row-oriented RocksDB encoding.

Physical format changes are gated behind **format major versions**: a store
opens at its current format, and upgrades are an explicit, *permanent* ratchet
via `DB.RatchetFormatMajorVersion` — no downgrade path. Some migrations are
instant, some run in the background, some block until complete[^5].

## Production Notes

- **The README's own warning applies**: Pebble may *silently corrupt* a
  RocksDB database that used an unsupported feature (column families,
  plain/hash table formats, sstable format v3/v4). RocksDB migration was a
  `v1`-only path; step through format major versions before moving to `v2`.
- **Resource lifecycle is manual.** `Get` values are valid only until their
  `Closer` is closed; iterators and snapshots must be closed or they pin
  memtables and sstables — a slow-motion disk-usage incident.
- **Single-process only.** An embedded library with a file lock — one process
  per store, no client-server mode, no multi-process readers.
- **Tuning differs from RocksDB.** Only level-based compaction exists; the
  knobs that matter are block cache size, memtable size/count, L0 stall
  thresholds, and compaction concurrency. The block cache is
  reference-counted and manually managed — misuse shows up as leaks.
- **Observability is good**: `DB.Metrics()` exposes per-level compaction,
  read-amp, and WAL stats; nightly benchmarks are public[^4].
- **Version churn tracks CockroachDB's release train.** `v1` → `v2` broke the
  module path (`/v2`), and newer releases drop old format major versions. Old
  series do get patches (v1.1.x into 2025, v2.0.x into 2026).
- **Documentation is thin**: godoc, a `docs/` directory, and the source.
  Expect to read code beyond the basics.

## When to Use / When Not

**Use when:**
- You need an embedded, high-write-throughput ordered KV store in a pure-Go
  binary (no cgo, single static binary, easy cross-compilation).
- You want LSM strengths: fast sequential writes, range scans, prefix
  iteration, bloom-filtered point reads.
- You accept CockroachDB-grade rigor with a roadmap you don't control.

**Avoid when:**
- You need transactions, column families, or backups — RocksDB has them,
  Pebble deliberately does not.
- Your workload is read-heavy with rare writes — a B-tree store is simpler
  and avoids compaction entirely.
- You need multi-process access or a network protocol — this is a library.
- You want community-driven features; the issue tracker serves CockroachDB.

## Alternatives

- facebook/rocksdb — use instead when you need column families, transactions,
  backups, or a non-Go language; costs you cgo in Go programs.
- dgraph-io/badger — use instead for a community-oriented Go LSM with
  WiscKey-style key-value separation and a simpler API.
- etcd-io/bbolt — use instead for read-heavy workloads wanting B+tree
  simplicity, single-file stores, and ACID single-writer transactions.
- syndtr/goleveldb — use instead only for legacy compatibility; a pure-Go
  LevelDB port that geth and others have migrated away from.
- google/leveldb — use instead when you want the minimal C++ original.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-07 | Development begins, seeded from the incomplete golang/leveldb port. |
| — | 2020-05 | Optional storage engine in CockroachDB 20.1[^2]. |
| — | 2020-11 | Default storage engine in CockroachDB 20.2[^2]. |
| v1.0.0 | 2023-12-15 | First tagged stable release; RocksDB 6.2.1 forward-compatible. |
| v2.0.0 | 2025-01-06 | Drops RocksDB store compatibility; columnar block format[^5]. |
| v2.1.0 | 2025-08-27 | Current stable series; older series still receive patches. |
| v2.1.6 | 2026-05-27 | Latest patch release at time of writing. |

## References

[^1]: Cockroach Labs, "Pebble: A RocksDB-inspired key-value store written in Go" — 2020-09. https://www.cockroachlabs.com/blog/pebble-rocksdb-kv-store/
[^2]: Pebble README, "Production Ready". https://github.com/cockroachdb/pebble#production-ready
[^3]: go-ethereum documentation, "Databases" (Pebble backend). https://geth.ethereum.org/docs/fundamentals/databases
[^4]: "Pebble vs RocksDB: Implementation Differences". https://github.com/cockroachdb/pebble/blob/master/docs/rocksdb.md ; nightly benchmarks: https://cockroachdb.github.io/pebble/
[^5]: Pebble README, "Format major versions"; v2.0.0 release. https://github.com/cockroachdb/pebble/releases/tag/v2.0.0

## Tags

go, key-value-store, lsm-tree, storage-engine, embedded-database, rocksdb, cockroachdb, database-internals, compaction
