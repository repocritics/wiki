# allegro/bigcache

> An in-memory, sharded, GC-friendly cache for Go that holds gigabytes of entries by storing them as bytes the garbage collector never scans.

[GitHub repo](https://github.com/allegro/bigcache) ·
[Origin blog post](http://allegro.tech/2016/03/writing-fast-cache-service-in-go.html) ·
[License: Apache-2.0](https://github.com/allegro/bigcache/blob/main/LICENSE)

## Overview

BigCache is a concurrent in-process cache written at Allegro, a Polish e-commerce company, and released in March 2016 alongside a blog post describing the problem it was built for: a cache holding millions of entries whose garbage-collection pauses were unacceptable[^1]. The defining idea is that Go's GC, as of Go 1.5, skips scanning maps whose keys and values contain no pointers[^2]. BigCache exploits this by keeping the index as `map[uint64]uint32` (hashed key to byte offset) and packing the actual entries into large `[]byte` buffers. The GC sees a handful of pointers instead of millions, so pause time stays flat as the dataset grows into the gigabytes.

The tradeoff is baked into the API: values go in and come out as `[]byte`. Anything richer than a byte slice must be serialized before `Set` and deserialized after `Get`, and that (de)serialization cost is the caller's to pay and measure. BigCache is not a general-purpose object cache; it is a byte-slice store optimized for one property — bounded GC impact — at the expense of ergonomics.

It suits services that cache large volumes of already-serialized data (rendered JSON, protobuf blobs, HTML fragments) with a time-based eviction model. It is a poor fit for callers who want an LRU of live Go structs, per-key TTLs, or cache semantics beyond "expire after a fixed window." Those are deliberate omissions, not gaps to be filled.

## Getting Started

```bash
go get github.com/allegro/bigcache/v3
```

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/allegro/bigcache/v3"
)

func main() {
	cache, _ := bigcache.New(context.Background(), bigcache.DefaultConfig(10*time.Minute))

	cache.Set("my-key", []byte("value")) // values are always []byte

	entry, err := cache.Get("my-key")
	if err != nil {
		// bigcache.ErrEntryNotFound when missing or expired
		return
	}
	fmt.Println(string(entry))
}
```

`DefaultConfig(eviction)` takes only the `LifeWindow`. For predictable load, use the full `Config` to pre-size allocations (`MaxEntriesInWindow`, `MaxEntrySize`) and cap memory (`HardMaxCacheSize`, in MB) so the process cannot grow unbounded.

## Architecture / How It Works

The cache is split into **shards** (configurable, must be a power of two; default 1024). A key is hashed with FNV-1a, and the low bits select the shard, so lock contention is spread across shards rather than a single global mutex. Each shard owns its own `sync.RWMutex`, its own `map[uint64]uint32` index, and its own `[]byte` backing buffer (a "bytes queue").

Within a shard, entries are appended to the byte queue as length-prefixed records containing a timestamp, the key hash, the key, and the value. The index maps the hashed key to the entry's offset. Reads locate the offset, slice out the record, and verify the stored key hash. **This is the crucial caveat: BigCache does not resolve hash collisions.** If two distinct keys hash to the same `uint64`, the second `Set` overwrites the first, and a `Get` for the evicted key returns the wrong shard slot — mitigated only by the stored-key comparison, which turns a collision into a miss rather than corrupt data[^3].

Eviction is time-based, driven by two windows. `LifeWindow` marks when an entry is considered dead; `CleanWindow` is how often a background goroutine sweeps dead entries out of the queue. There is no LRU and no per-entry TTL — every entry in a cache shares the same lifetime. When `HardMaxCacheSize` is hit, the oldest entries are evicted FIFO to make room, regardless of whether they are still within their life window. The optional `OnRemove` / `OnRemoveWithReason` callbacks fire on eviction, expiry, or delete, with a reason code distinguishing the three.

The `server/` subpackage wraps the cache in a small HTTP service, and a separate `allegro/bigcache-bench` repo hosts the comparison benchmarks against freecache and a plain map.

## Production Notes

- **Everything is `[]byte`.** Budget for serialization on both sides. For small structs the marshal/unmarshal cost can dominate the cache's own sub-microsecond access time, erasing the benefit versus a plain `sync.Map` for modest datasets.
- **Shard count is a tuning knob, not a default to ignore.** Too few shards under high concurrency reintroduces lock contention; too many wastes memory on per-shard buffers. Size shards against real goroutine concurrency.
- **Memory is reserved, not returned.** Byte queues grow and stay allocated; the process RSS climbs and plateaus rather than shrinking after eviction. The README explicitly warns that apparent "exponential" memory growth is the Go runtime holding idle spans, not a leak[^4]. Set `HardMaxCacheSize` if the host has hard memory limits.
- **`CleanWindow` has one-second resolution.** Setting it below one second is counterproductive; the cleanup goroutine cannot act more finely.
- **Uniform TTL only.** If you need different lifetimes per entry, BigCache cannot express it — you would run multiple caches or encode expiry into the value and filter on read.
- **Collisions silently drop entries.** With a 64-bit hash the probability is low but non-zero at very large entry counts; treat the cache as best-effort, never as a system of record.
- **Iteration exists but is a snapshot.** `Iterator()` walks entries but is not consistent under concurrent writes; use it for diagnostics, not correctness-critical scans.

## When to Use / When Not

**Use when:**
- You cache large numbers (millions) of already-serialized byte payloads and GC pause time is a measured problem.
- A single shared time-to-live per cache is acceptable.
- You want an in-process cache with no external service (unlike Redis/Memcached) and no cgo.

**Avoid when:**
- You want to cache live Go objects without paying serialization cost — reach for a struct-valued LRU.
- You need per-key TTLs, LRU/LFU eviction, or cache-wide consistency guarantees.
- Your dataset is small enough that plain `map` + `sync.RWMutex` or `sync.Map` has negligible GC impact; BigCache's byte-slice tax is not worth it.
- You need cross-process or distributed caching — this is single-process only.

## Alternatives

- coocood/freecache — same GC-avoidance goal via a ring buffer; requires sizing the cache up front and overwrites rather than growing when full.
- dgraph-io/ristretto — admission-controlled (TinyLFU) cache with hit-ratio focus and generic values; use when eviction quality matters more than raw GC avoidance.
- hashicorp/golang-lru — simple, correct LRU of arbitrary values; use for bounded object caches where GC pressure is not the constraint.
- patrickmn/go-cache — map-backed cache with per-item TTL; use when you need per-key expiry and hold far fewer entries.
- redis/go-redis — client for an external Redis instance; use when the cache must be shared across processes or hosts.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2016-03-23 | Published with the "fast cache service in Go" blog post[^1]. |
| v1.x | 2016–2018 | Sharded byte-queue design, time-window eviction stabilize. |
| v2.0.0 | 2019 | Module path `bigcache/v2`; API refinements. |
| v3.0.0 | 2021 | `bigcache/v3`; `context.Context` added to `New`, current major line[^5]. |

As of mid-2026 the project remains maintained (last push June 2026) with roughly 8,100 stars and 600 forks — a mature, low-churn library that sees maintenance and dependency updates rather than large feature swings.

## References

[^1]: Łukasz Druminski, "Writing a very fast cache service with millions of entries in Go" — allegro.tech, 2016-03-23. http://allegro.tech/2016/03/writing-fast-cache-service-in-go.html
[^2]: Go issue #9477, "runtime: GC does not scan pointer-free maps." https://github.com/golang/go/issues/9477
[^3]: BigCache README, "Collisions" section. https://github.com/allegro/bigcache#collisions
[^4]: BigCache README, "Memory usage" section, citing Go runtime span behavior. https://github.com/allegro/bigcache
[^5]: BigCache module path `github.com/allegro/bigcache/v3`. https://pkg.go.dev/github.com/allegro/bigcache/v3

## Tags

go, cache, in-memory-cache, gc-optimization, sharding, key-value, performance, concurrency, byte-slice, library
