# google/leveldb

> An embedded, ordered key-value store built on a log-structured merge-tree — the code that seeded RocksDB and a generation of storage engines.

[GitHub repo](https://github.com/google/leveldb) ·
[License: BSD-3-Clause](https://github.com/google/leveldb/blob/main/LICENSE)

## Overview

LevelDB is a C++ library that maps arbitrary byte-string keys to byte-string
values, keeping keys sorted, with atomic batched writes, forward/backward
iteration, and consistent point-in-time snapshots. It was written at Google by
Sanjay Ghemawat and Jeff Dean, drawing directly on the design of Bigtable's
tablet storage, and open-sourced in 2011[^1]. The GitHub mirror was created in
2014 when the project moved off Google Code.

It is a library, not a database server. There is no network layer, no query
language, no relational model, and no built-in indexing beyond the single
sorted keyspace. Exactly one process may open a database at a time (enforced by
a `LOCK` file); concurrency is intra-process across threads. This narrow scope
is deliberate: LevelDB is meant to be the storage engine underneath something
else, and it shows up embedded in Chromium's IndexedDB, Bitcoin Core, and
countless application backends.

The defining tradeoff is the LSM-tree itself: writes are fast and sequential
because they append to a log and an in-memory table, but that speed is repaid
later through background compaction, which reads and rewrites data across disk
levels (write amplification) and can stall foreground writes when it falls
behind. LevelDB optimizes for write throughput and range scans at the cost of
worst-case read latency and steady background I/O. As of 2026 the repository is
explicitly under "very limited maintenance" — only critical-bug and
compatibility fixes are reviewed[^2] — which is a statement about stability, not
abandonment: the code is considered largely done, and active evolution lives in
its forks.

## Getting Started

LevelDB builds with CMake and vendors its test dependencies as submodules:

```bash
git clone --recurse-submodules https://github.com/google/leveldb.git
cd leveldb
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```

A minimal open / put / get / iterate cycle:

```cpp
#include "leveldb/db.h"
#include "leveldb/write_batch.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::DB::Open(options, "/tmp/testdb", &db);   // one process at a time

leveldb::WriteBatch batch;                          // atomic multi-mutation
batch.Put("name", "leveldb");
batch.Delete("stale-key");
db->Write(leveldb::WriteOptions(), &batch);

std::string value;
db->Get(leveldb::ReadOptions(), "name", &value);    // value == "leveldb"

leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {  // ordered scan
  // it->key(), it->value() are Slices into internal buffers.
}
delete it;
delete db;
```

## Architecture / How It Works

LevelDB is a canonical LSM-tree. A write goes through two structures before it
reaches its final resting place:

1. **Write-ahead log** — every `Put`/`Delete` is first appended to an on-disk
   log file for durability. `WriteOptions::sync` controls whether the OS is
   asked to flush to the platter; the default is unsynced (fast, but a power
   loss can lose the last batches).
2. **MemTable** — an in-memory sorted skiplist. Writes land here after the log
   append. When it exceeds `write_buffer_size` (default 4 MB) it becomes an
   immutable memtable and a fresh one takes over.

A background thread flushes the immutable memtable to a **sorted string table
(SSTable / `.ldb`)** — an immutable file of sorted key-value blocks plus an
index and an optional Bloom filter. SSTables live in numbered **levels**.
Level-0 files can have overlapping key ranges (they come straight from
memtables); levels 1 through 6 are non-overlapping, and each level is roughly
ten times larger than the one above. **Compaction** merges files from level N
into level N+1, discarding overwritten and deleted keys.

Reads consult, in order: the mutable memtable, the immutable memtable, then each
level's SSTables (all of level 0, then one file per lower level via binary
search on the index). **Bloom filters** let a read skip SSTables that provably
lack the key, which is what keeps point lookups from degrading linearly with
level count. Blocks are cached uncompressed in an LRU **block cache** to avoid
repeated decompression.

Ordering and versioning ride on a monotonically increasing **sequence number**
attached to every internal key. **Snapshots** pin a sequence number so an
iterator sees a consistent view without blocking writers — MVCC without
explicit undo logs. The `VersionSet`, the `MANIFEST`, and the `CURRENT` file
together track which SSTables constitute the live database at any moment.

Two abstractions make the engine adaptable: the **`Comparator`** (pluggable key
ordering) and the **`Env`** (every filesystem and threading call routes through
this virtual interface, so callers can supply in-memory or instrumented I/O).
Values are compressed per-block with Snappy by default; Zstd is also supported.

## Production Notes

- **Single-process, single-open.** A database can be opened by exactly one
  process. There is no shared-cache or multi-process mode. Attempting a second
  open fails on the `LOCK` file. Design your process topology around this before
  you build on it.
- **Write stalls are real and abrupt.** When level-0 accumulates too many files,
  LevelDB first slows writers (a ~1 ms delay per write) and then stops them
  entirely until compaction catches up. Sustained write bursts that outrun the
  single background compaction thread manifest as latency cliffs, not gradual
  degradation. This is the number-one operational surprise.
- **Write amplification.** Leveled compaction rewrites data many times as it
  migrates downward; total bytes written to disk can be an order of magnitude
  larger than bytes the application wrote. On write-heavy workloads over
  flash, factor this into device endurance and IOPS budgeting.
- **`fillsync` is slow by design.** Synchronous writes wait on a disk flush
  (hundreds of microseconds to milliseconds each). High-durability workloads
  should batch many mutations into one `WriteBatch` per sync rather than syncing
  per key.
- **Snapshots are in-memory only.** They are not durable checkpoints. There is
  no built-in backup, replication, or hot-copy; you cannot restore a snapshot
  after reopening. Backups mean copying the closed database directory yourself.
- **Corruption recovery is manual.** `leveldb::RepairDB` salvages what it can
  from damaged files, but it is lossy. Enable `paranoid_checks` if you want the
  library to surface corruption early rather than silently.
- **Tuning knobs are few.** Compared to its forks, LevelDB exposes little:
  `write_buffer_size`, `block_cache`, `max_open_files`, `block_size`, and a
  filter policy. There are no column families, no transactions beyond atomic
  `WriteBatch`, and no built-in multi-threaded compaction. Workloads that need
  those graduated to RocksDB long ago.

## When to Use / When Not

**Use when:**
- You need a small, dependency-light embedded ordered store inside a single
  process (desktop app state, browser storage, a local cache, a blockchain
  node's index).
- Your access pattern is write-heavy with range scans and you can tolerate
  background compaction I/O.
- You want proven, stable C++ code with a tiny, unchanging API surface.

**Avoid when:**
- Multiple processes must read/write the same data — you need a server (or
  RocksDB with a wrapper).
- You need transactions across keys, column families, a rich tuning surface,
  or multi-threaded compaction — reach for RocksDB.
- You need SQL, secondary indexes, or a relational model — use SQLite.
- Reads dominate overwhelmingly and write amplification / compaction stalls are
  unacceptable — a B+tree store (LMDB, bbolt) may fit better.

## Alternatives

- facebook/rocksdb — the dominant fork of LevelDB; adds column families,
  transactions, multi-threaded compaction, and a deep tuning surface. Use when
  you need production-grade knobs on the same LSM model.
- symas/lmdb — memory-mapped B+tree with zero compaction and copy-on-write MVCC.
  Use when reads dominate and you want no background write amplification.
- etcd-io/bbolt — a pure-Go single-file B+tree store. Use when you're in Go and
  want a read-optimized embedded engine with a simple file format.
- dgraph-io/badger — Go LSM engine that separates keys from values (WiscKey).
  Use for Go services storing large values.
- sqlite/sqlite — use when you actually want a relational model, SQL, and
  secondary indexes rather than a raw ordered map.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011-03 | Open-sourced by Google on Google Code[^1]. |
| 1.4 | 2012 | Bloom filter support added, cutting point-read I/O. |
| — | 2012 | Facebook forks LevelDB to start RocksDB. |
| 1.18 | 2014 | Repository mirrored to GitHub as google/leveldb. |
| 1.20 | 2017-03 | Release with build and portability fixes. |
| 1.21 | 2019-05 | Build system moved to CMake; project layout modernized. |
| 1.22 | 2019-05 | Maintenance and platform fixes. |
| 1.23 | 2021-02 | Latest tagged release; ongoing critical-fix-only mode[^2]. |

## References

[^1]: Google Open Source Blog, "LevelDB: A Fast Persistent Key-Value Store" — 2011-07-27. https://opensource.googleblog.com/2011/07/leveldb-fast-persistent-key-value-store.html
[^2]: google/leveldb README — "This repository is receiving very limited maintenance." https://github.com/google/leveldb/blob/main/README.md
[^3]: LevelDB implementation notes (doc/impl.md). https://github.com/google/leveldb/blob/main/doc/impl.md

## Tags

key-value-store, embedded-database, lsm-tree, storage-engine, cpp, database, ordered-map, snappy, google, no-sql
