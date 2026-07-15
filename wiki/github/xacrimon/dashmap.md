# xacrimon/dashmap

> A sharded concurrent HashMap for Rust that swaps in for `RwLock<HashMap>` and takes `&self` on every method.

[GitHub repo](https://github.com/xacrimon/dashmap) ·
[docs.rs](https://docs.rs/dashmap) ·
[License: MIT](https://github.com/xacrimon/dashmap/blob/master/LICENSE)

## Overview

DashMap is a concurrent associative array for Rust, first published in late 2019[^1]. Its design goal is narrow and explicit: be a drop-in replacement for `RwLock<HashMap<K, V>>` that removes the need to thread a single big lock through your code. Every mutating method takes `&self` rather than `&mut self`, so a `DashMap` can sit inside an `Arc` and be shared across threads while still being written to. This is the entire value proposition — ergonomics and throughput over a single global lock — and it is delivered by internal sharding rather than any lock-free algorithm.

The library is maintained primarily by one author (Joel Wejdenstam) and is candid that the work is unpaid and time-limited[^1]. Despite that, it is one of the most widely depended-on concurrency crates in the Rust ecosystem, pulled in transitively by a large fraction of async and server code. As of this writing it has roughly 4.1k stars and 187 forks, with the last push in May 2026[^2] — steady, mature maintenance rather than rapid feature development.

The defining tension is that DashMap's ergonomics hide a lock. The `&self` API and the guard types (`Ref`/`RefMut`) make it feel like a lock-free map, but under the hood you are holding a shard-level `RwLock`. Code that would obviously deadlock with an explicit lock still deadlocks here — it is just less visually obvious. Understanding the sharding model is the difference between using DashMap safely and shipping intermittent hangs.

## Getting Started

```bash
cargo add dashmap
```

```rust
use dashmap::DashMap;
use std::sync::Arc;

let map: Arc<DashMap<String, i32>> = Arc::new(DashMap::new());

// Share across threads — no external Mutex/RwLock needed.
let writer = Arc::clone(&map);
let handle = std::thread::spawn(move || {
    writer.insert("hits".to_string(), 1);
});
handle.join().unwrap();

// get() returns a Ref guard holding a read lock on the key's shard.
if let Some(v) = map.get("hits") {
    println!("{}", *v);
}

// entry() takes the shard's write lock for the duration of the call.
map.entry("hits".to_string())
    .and_modify(|v| *v += 1)
    .or_insert(0);
```

## Architecture / How It Works

DashMap is, at its core, an array of shards: `[RwLock<HashMap<K, V>>; N]`, where each inner map is a `hashbrown` table (the same SwissTable implementation `std::collections::HashMap` uses). A key is hashed once; the top bits of the hash select which shard it belongs to, and the remaining bits are reused to index within that shard's table. The number of shards defaults to a multiple of the detected CPU count rounded up to a power of two, so parallel access to keys that land on different shards proceeds without contending on the same lock.

Reads and writes acquire a lock on **one shard**, not one entry. `get` takes the shard's read lock and hands back a `Ref` guard; `get_mut` and `entry` take the write lock and hand back a `RefMut`. The lock is held for as long as the guard is alive. This is the single most important fact about DashMap: the guard is a live lock, and its lifetime is your responsibility.

Iteration (`iter`, `iter_mut`) walks shards one at a time, locking each as it goes. Consequently a full iteration is **not** an atomic snapshot of the map — writes committed to a shard you have already passed, or one you have not yet reached, may or may not be visible. There is no global lock and no MVCC; consistency is per-shard only.

The crate is deliberately not lock-free. It uses no atomics-based hazard pointers, epochs, or RCU — the concurrency story is entirely "many small `RwLock`s." An unstable `raw-api` feature exposes the shard array directly for callers who want to lock a shard manually. Optional features add `serde`, `rayon` parallel iteration, `arbitrary`, and `inline-more` (forwarding hashbrown's aggressive inlining, with its usual code-size tradeoff)[^1].

## Production Notes

**The deadlock footgun is real and the top cause of reported hangs.** Because a guard holds a shard lock, holding one guard while accessing another key that happens to hash to the same shard will deadlock — for example, calling `get_mut` (write lock) and then `get` on the same shard, or holding any `Ref`/`RefMut` across a second write. You cannot predict which keys share a shard, so the only safe rule is: never hold a `Ref` or `RefMut` across another operation on the same map. Drop the guard first[^3]. This is the DashMap equivalent of "don't lock the same mutex twice," but the `&self` API makes it easy to forget you hold a lock at all.

**Hot-shard contention.** Sharding distributes load statistically, not perfectly. A workload dominated by a few hot keys will serialize on their shards regardless of how many total shards exist. DashMap does not help — and can hurt versus a purpose-built cache — when access is heavily skewed.

**`iter()` under concurrent mutation.** Long-running iterations hold each shard's lock while you process its entries, so a slow closure inside `for entry in map.iter()` blocks writers to that shard. Keep per-entry work short, or collect keys and re-fetch. Also remember iteration is not a consistent snapshot.

**MSRV policy.** The crate pins a minimum supported Rust version (1.70 at time of writing) and intentionally trails current stable by at least a year; MSRV is not bumped in patch releases, so you can pin a minor version for stability[^1]. This makes it safe for conservative toolchains but means it will not adopt the newest language features quickly.

**Value cloning.** `get` returns a guard, not a clone, so reading is cheap — but any pattern that clones values out to avoid holding the guard trades the deadlock risk for allocation cost. For `Copy` values this is free; for large owned values, weigh it.

**Upgrade friction.** Major versions (notably the 4→5 and 5→6 transitions) changed default hashers and guard/API surface; a bump is usually a small but non-mechanical migration rather than a drop-in, so read the changelog before bumping across a major.

## When to Use / When Not

**Use when:**
- You have a genuinely shared, concurrently-mutated map and want to delete an `RwLock<HashMap>` and its borrow-checker friction.
- Access is spread across many keys (good shard distribution) with a mix of reads and writes.
- You want a `std`-like API (`insert`, `get`, `entry`, `remove`) with no async runtime dependency.

**Avoid when:**
- Reads vastly outnumber writes and you need lock-free reads — a read-optimized structure will scale better.
- Access is skewed to a few hot keys — sharding does not solve contention on them.
- You need atomic multi-key operations, consistent snapshots, ordering, or eviction/TTL — these are out of scope.
- Your "concurrency" is a single thread or a short critical section — a plain `HashMap` behind a `Mutex` is simpler and often faster.

## Alternatives

- crossbeam-rs/crossbeam — crossbeam-skiplist gives an ordered, lock-free concurrent map; use it when you need sorted iteration or lock-free reads more than raw hashmap speed.
- jonhoo/flurry — a port of Java's ConcurrentHashMap with lock-free reads via guards; use when reads dominate and you can tolerate the guard/epoch API.
- jonhoo/left-right — evmap, an eventually-consistent read-optimized map using double buffering; use when reads massively outnumber writes and slight staleness is acceptable.
- wvwwvwwv/scalable-concurrent-containers — scc's HashMap targets better tail latency under heavy contention and offers async variants; use when DashMap's shard contention shows up in profiles.
- moka-rs/moka — a concurrent cache with eviction and TTL; use when you actually want a bounded cache, not a bare map.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-12-06 | First published; sharded `RwLock<HashMap>` design[^2]. |
| 4.x | 2021 | Widely adopted major line; hashbrown-backed shards. |
| 5.0 | 2022 | Major release; hasher/default and API changes, guard refinements[^4]. |
| 6.0 | 2024 | Latest major line; continued API and internals cleanup[^4]. |

## References

[^1]: DashMap README — features, MSRV policy, maintainer note. https://github.com/xacrimon/dashmap/blob/master/README.md
[^2]: GitHub API, `repos/xacrimon/dashmap` — stars, forks, created/pushed dates, license (fetched for this page). https://github.com/xacrimon/dashmap
[^3]: DashMap deadlock semantics discussion (guards hold shard locks). https://github.com/xacrimon/dashmap/issues
[^4]: crates.io release history and changelog for the `dashmap` crate. https://crates.io/crates/dashmap/versions

## Tags

rust, concurrency, hashmap, data-structures, concurrent-map, sharding, lock-based, thread-safe, crates-io
