# hashicorp/golang-lru

> Fixed-size, thread-safe LRU cache for Go — the default in-memory eviction primitive across the HashiCorp stack and much of the wider Go ecosystem.

[GitHub repo](https://github.com/hashicorp/golang-lru) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/hashicorp/golang-lru/v2) ·
[License: MPL-2.0](https://github.com/hashicorp/golang-lru/blob/main/LICENSE)

## Overview

`golang-lru` is a small, single-purpose library: a fixed-capacity cache that evicts the least-recently-used entry when it fills up. The core eviction logic is a direct descendant of the LRU cache in Google's Groupcache[^1] — a doubly-linked list for recency ordering plus a map for O(1) lookup — with a mutex wrapper added for concurrent use. It is not a distributed cache, not a TTL store by default, and not a memory-bounded (byte-size) cache; capacity is counted in number of entries.

The library's reach is out of proportion to its size. It ships inside Consul, Vault, Nomad, and Terraform, and through them sits in the transitive dependency tree of a large share of production Go services[^2]. That ubiquity is the reason to treat it as infrastructure rather than a casual import: its API shape, its MPL-2.0 license, and one patent footgun (the ARC variant, see Production Notes) have all propagated widely.

The defining tension is scope discipline versus feature creep. The maintainers have kept the hot path deliberately simple — one lock, one list, one map — and pushed everything else (TTL expiry, 2Q, ARC) into separate types or, later, separate Go modules. This keeps the common case fast and auditable, at the cost of making you choose the right variant up front.

## Getting Started

```bash
go get github.com/hashicorp/golang-lru/v2
```

```go
package main

import (
	"fmt"

	lru "github.com/hashicorp/golang-lru/v2"
)

func main() {
	// Capacity is entry count, not bytes. New errors only on size <= 0.
	cache, _ := lru.New[string, int](128)

	evicted := cache.Add("a", 1) // evicted == true if this pushed one out
	_ = evicted

	if v, ok := cache.Get("a"); ok { // Get promotes "a" to most-recent
		fmt.Println(v) // 1
	}

	cache.Contains("a") // membership check WITHOUT promoting recency
	cache.Peek("a")     // read value WITHOUT promoting recency
	fmt.Println(cache.Len())
}
```

For time-based expiry, use the separate `expirable` package:

```go
import "github.com/hashicorp/golang-lru/v2/expirable"

// 5 entries max, per-entry TTL of 10ms, nil onEvict callback.
c := expirable.NewLRU[string, string](5, nil, 10*time.Millisecond)
```

## Architecture / How It Works

There are three layers, and knowing which one you touch matters:

1. **`simplelru`** — the non-thread-safe core. A `container/list` doubly-linked list holds entries in recency order; a `map[K]*list.Element` gives O(1) access. `Add`, `Get`, `Remove`, `RemoveOldest` are all constant time. It takes no lock, so you can embed it under your own synchronization.
2. **`lru`** (the root/v2 package) — wraps `simplelru` in a `sync.RWMutex`. Note that even `Get` takes the *write* lock, because a read promotes the entry to most-recently-used and mutates the list. Under read-heavy, high-contention workloads this single lock is the scaling ceiling.
3. **Specialized variants** — `TwoQueueCache` (2Q), the ARC cache, and `expirable.LRU`. 2Q and ARC add scan-resistance by tracking recent-but-evicted keys; `expirable` adds TTL with lazy plus bucketed expiration.

Generics are the major structural fact of the modern library. The `/v2` module (Go 1.18+) replaced `interface{}` keys and values with type parameters `[K comparable, V any]`[^3], removing per-operation boxing and type assertions. The pre-generics `v1` API still exists at the root import path (no `/v2`) for code that cannot upgrade.

The `Add` method returns a boolean indicating whether an eviction occurred; eviction callbacks are registered via `NewWithEvict`. Because eviction can fire synchronously inside `Add` while the lock is held, an expensive or blocking `onEvict` callback stalls every other cache operation — a common source of latency surprises.

## Production Notes

**The ARC patent footgun.** The Adaptive Replacement Cache algorithm is covered by a US patent historically held by IBM, and the source has long carried a comment warning that using the ARC cache "may require a license"[^4]. This is why ARC was extracted into its own Go module (`golang-lru/arc/v2`) rather than living in the main package — so that a `go get` of the main library does not silently pull the patented algorithm into your build. If you need scan resistance without the legal question, prefer `TwoQueueCache`, which is not encumbered.

**One lock, one hot path.** The RWMutex is effectively a plain mutex for cache traffic because `Get` mutates recency. On many-core machines with high read concurrency this contends. The standard mitigations are sharding the cache yourself (N sub-caches keyed by a hash of the key) or switching to a library built for concurrency (Ristretto, otter). Do not expect the built-in cache to scale linearly with cores.

**Capacity is entries, not memory.** A 128-entry cache of 10 MB blobs is 1.3 GB. The library has no byte-size accounting; if entry sizes vary you must bound memory yourself or pick a size-aware cache.

**`expirable` TTL semantics.** TTL is uniform per cache (set at construction), not per entry, and expiration is a mix of lazy checks on access plus periodic bucket cleanup — an entry can remain resident (and count toward capacity) past its TTL until it is touched or swept. It is not a precise timer.

**Callbacks run under the lock.** `onEvict` executes synchronously inside the critical section. Keep it to a metric increment or a channel send; never do I/O there.

**Maintenance cadence.** The library is stable rather than actively evolving: the last tagged release (v2.0.7) is from 2023, though the default branch still receives occasional commits. For a primitive this widely deployed, "finished" is a reasonable read — but do not expect new features.

## When to Use / When Not

**Use when:**
- You need a correct, boring, entry-bounded LRU in a single process.
- Your workload is not lock-contention-bound (moderate concurrency, or you shard yourself).
- You want a dependency that is already audited and vendored across the Go ecosystem.
- You need simple 2Q/ARC scan resistance and can pick the right variant.

**Avoid when:**
- You need maximum throughput under heavy multi-core read contention — reach for a sharded/lock-optimized cache instead.
- You must bound by bytes, not entry count.
- You need precise per-entry TTL expiry (the `expirable` variant is coarse).
- You want ARC but cannot accept the patent/licensing ambiguity.

## Alternatives

- dgraph-io/ristretto — use instead when you need high-concurrency throughput and hit-ratio optimization (TinyLFU admission, sharded, byte-cost aware).
- maypok86/otter — use when you want a modern eviction policy (S3-FIFO) that beats classic LRU on hit ratio while staying fast under contention.
- karlseguin/ccache — use when you want LRU plus real per-item TTL together in one GC-friendly structure.
- jellydator/ttlcache — use when time-based expiry is the primary requirement and LRU eviction is secondary.
- coocood/freecache — use when GC pressure from millions of entries is the actual problem (off-heap, zero-GC storage).

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-08 | First release; LRU derived from Groupcache[^1]. |
| v0.5.x  | ~2018–2021 | 2Q and ARC caches added alongside the base LRU. |
| v0.6.0  | 2022-11-14 | Final pre-generics `interface{}` release line. |
| v2.0.0  | 2022-11-14 | Generics rewrite; `/v2` module, typed keys/values[^3]. |
| v1.0.1  | 2022-11-15 | Backport line for pre-generics consumers. |
| v2.0.2  | 2023-03-13 | `expirable` TTL cache added to the v2 line. |
| v2.0.7  | 2023-09-29 | Latest tagged release; ARC lives in its own module. |

## References

[^1]: Groupcache LRU, the ancestor implementation. https://github.com/golang/groupcache/tree/master/lru
[^2]: Package importers on pkg.go.dev show adoption across the HashiCorp stack and beyond. https://pkg.go.dev/github.com/hashicorp/golang-lru/v2?tab=importedby
[^3]: golang-lru v2.0.0 release (generics, `/v2` module path). https://github.com/hashicorp/golang-lru/releases/tag/v2.0.0
[^4]: ARC cache implementation and its IBM-patent licensing note. https://github.com/hashicorp/golang-lru/blob/main/arc/arc.go

## Tags

go, cache, lru, in-memory-cache, eviction, generics, concurrency, hashicorp, data-structures, mpl-2.0
