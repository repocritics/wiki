# tidwall/buntdb

> An embeddable, in-memory, ACID key/value store in pure Go, with B-tree, spatial (R-tree), and JSON indexes.

[GitHub repo](https://github.com/tidwall/buntdb) ·
[License: MIT](https://github.com/tidwall/buntdb/blob/master/LICENSE)

## Overview

BuntDB is a low-level embeddable database written by Josh Baker (tidwall), the
author of Tile38, GJSON, and a broader family of Go data libraries[^1]. It keeps
the entire dataset in memory, persists to disk with an append-only log, and
exposes an ACID transaction API where all reads and writes happen inside
`db.View` / `db.Update` closures. It is a library, not a server: there is no
network protocol, no query language, and no client/server split — you import it
and call functions.

The defining tradeoff is stated plainly in the project's own description:
BuntDB favors speed over data size[^2]. Because every key and value lives in RAM
(and indexes add more), the working set is bounded by available memory. In
exchange you get microsecond-scale reads, ordered iteration over a B-tree, and
secondary indexes — including a genuine R-tree for geospatial queries — that
most embedded key/value stores do not offer. It sits between a raw
`map[string]string` and a full embedded engine like BoltDB or Badger: richer
querying than the former, smaller footprint and simpler model than the latter,
with the hard memory ceiling as the cost.

Concurrency is single-writer / multiple-reader, enforced with a mutex. One
read/write transaction runs at a time; many read-only transactions run
concurrently. This is simple to reason about and cheap for read-heavy workloads,
but it makes write throughput a serialization point.

## Getting Started

```sh
go get -u github.com/tidwall/buntdb
```

```go
package main

import (
	"fmt"
	"log"

	"github.com/tidwall/buntdb"
)

func main() {
	// Use a file path to persist; ":memory:" for a non-persistent store.
	db, err := buntdb.Open("data.db")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Write in an Update transaction (returning an error rolls back).
	err = db.Update(func(tx *buntdb.Tx) error {
		_, _, err := tx.Set("user:1:name", "tom", nil)
		return err
	})
	if err != nil {
		log.Fatal(err)
	}

	// Read in a concurrent-safe View transaction.
	err = db.View(func(tx *buntdb.Tx) error {
		val, err := tx.Get("user:1:name")
		if err != nil {
			return err
		}
		fmt.Println(val) // tom
		return nil
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

## Architecture / How It Works

All primary data lives in a single B-tree keyed by the item key, giving ordered
iteration (`Ascend*` / `Descend*` variants) and fast point lookups[^3]. Values
are opaque strings; BuntDB never parses them unless you attach an index.

Secondary indexes are the distinguishing feature. `CreateIndex(name, pattern,
less...)` builds an additional B-tree ordered by a user-supplied comparison
function, populated only from keys matching a glob `pattern` (e.g. `user:*`).
Built-in comparators cover `IndexString`, `IndexInt`, `IndexUint`, `IndexFloat`,
and `IndexJSON("field.path")`, the last using GJSON to extract a field from JSON
values[^4]. Multiple comparators can be chained for multi-column ordering, and
any comparator can be wrapped with `Desc` for descending order.

Geospatial support uses a separate R-tree via `CreateSpatialIndex` and the
`IndexRect` parser, supporting up to 20 dimensions. Queries include
`Intersects` (bounding-box) and `Nearby` (k-nearest-neighbors). Rectangles use a
bracket syntax (`[-117 30],[-112 36]`) unique to BuntDB; longitude is the Y axis
and latitude is the X axis, a convention that trips up first-time users[^2].

Persistence is an append-only file (AOF): every `Set`/`Delete` is logged as a
text command and replayed on open. A background routine shrinks the log when it
grows past a configurable threshold (`AutoShrinkPercentage`, default 100%;
`AutoShrinkMinSize`, default 32MB), and `Shrink()` can be called manually without
blocking transactions. Durability is governed by `SyncPolicy`: `Never`,
`EverySecond` (default — up to one second of writes may be lost on crash), or
`Always` (fsync per write, durable but slower)[^2].

## Production Notes

- **Memory is the hard ceiling.** The full dataset plus every index resides in
  RAM. There is no spill-to-disk, no LRU of cold data (only TTL eviction). Size
  capacity planning around peak in-memory footprint, not disk.
- **Writes serialize.** A single write transaction at a time means write-heavy
  or high-contention workloads bottleneck on the writer lock. The published
  benchmarks show reads roughly an order of magnitude faster than writes[^5];
  design around read-mostly access patterns.
- **Default durability loses up to 1s of writes.** `EverySecond` is the default.
  If you cannot tolerate any loss on power failure, set `SyncPolicy: Always` and
  accept the per-write fsync cost. Many users are surprised by this default.
- **No delete-during-iteration.** The iterator does not support deleting the
  current key mid-scan; the documented workaround is to collect keys into a slice
  during iteration and delete them afterward[^2].
- **AOF replay on startup is O(operations logged).** For a large or un-shrunk
  log, open time scales with the number of historical commands, not the number
  of live keys — keep auto-shrink enabled and/or call `Shrink()` to bound this.
- **Indexes are rebuilt in memory on open**, adding to both startup time and
  resident memory; heavy secondary indexing multiplies the RAM cost.
- **Not a multi-process store.** One process owns the file. There is no shared
  access, no replication, and no clustering built in — for distributed use it is
  typically embedded behind something like a Raft log (the author's
  `raft-buntdb` is a reference for that pattern).

## When to Use / When Not

**Use when:**
- Your dataset comfortably fits in RAM and you want ordered iteration plus
  secondary/JSON/spatial indexes from a single small dependency.
- Reads dominate writes and you want microsecond point/range queries.
- You need embedded geospatial (R-tree `Nearby`/`Intersects`) without pulling in
  a full GIS stack.
- You want a pure-Go dependency with no cgo and a small, auditable codebase.

**Avoid when:**
- Your data exceeds memory or grows unbounded — there is no disk-backed cold tier.
- You have high write concurrency; the single-writer lock will serialize you.
- You need multi-process access, replication, or a network protocol out of the box.
- You need strong crash durability by default without tuning `SyncPolicy`.

## Alternatives

- etcd-io/bbolt — pure-Go B+tree, memory-mapped, disk-backed; use when data must
  exceed RAM and you want a single-file ACID store without indexes.
- dgraph-io/badger — LSM-tree key/value store; use when you need high write
  throughput and larger-than-memory datasets with SSD-friendly IO.
- cockroachdb/pebble — RocksDB-style LSM engine in Go; use when you want a
  production storage engine tuned for large, write-heavy workloads.
- tidwall/tile38 — same author's networked geospatial database (a server, Redis
  protocol); use when you need geospatial as a standalone service rather than a
  library.
- hashicorp/go-memdb — in-memory, immutable-radix, multi-index with MVCC; use
  when you want richer transactional multi-indexing and can stay in memory.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-07-19 | Repository created; extracted from the author's Go data-store work[^1]. |
| v1.0.0 | 2018-01-12 | First tagged release. |
| v1.1.0 | 2018-05-03 | Index and iteration API maturation. |
| v1.2.0 | 2021-02-08 | Ongoing refinements to indexing/transactions. |
| v1.3.0 | 2023-04-13 | Continued 1.3 line. |
| v1.3.2 | 2024-09-10 | Latest tagged release as of this writing. |

The project is mature and low-churn: the API has been stable across the 1.x
line, and while the repository still receives commits (most recent in
mid-2026), tagged releases are infrequent — a maintenance-mode posture rather
than active feature development.

## References

[^1]: BuntDB repository and author profile (Josh Baker, tidwall). https://github.com/tidwall/buntdb
[^2]: BuntDB README — features, spatial bracket syntax, config, durability, delete-while-iterating. https://github.com/tidwall/buntdb/blob/master/README.md
[^3]: BuntDB GoDoc / package reference. https://pkg.go.dev/github.com/tidwall/buntdb
[^4]: GJSON, used by BuntDB for JSON field indexing. https://github.com/tidwall/gjson
[^5]: BuntDB benchmark utility and reported operations/sec. https://github.com/tidwall/buntdb-benchmark

## Tags

go, golang, embedded-database, key-value-store, in-memory, geospatial, r-tree, b-tree, json-index, acid, append-only-log
