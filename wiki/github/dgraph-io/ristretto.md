# dgraph-io/ristretto

> A concurrent, cost-aware in-process Go cache that trades strict Set guarantees for high throughput and hit ratio.

[GitHub repo](https://github.com/dgraph-io/ristretto) ·
[Blog / homepage](https://hypermode.com/blog/introducing-ristretto-high-perf-go-cache) ·
[License: Apache-2.0](https://github.com/dgraph-io/ristretto/blob/main/LICENSE)

## Overview

Ristretto is an in-memory, single-process cache library for Go, built by the Dgraph team to give their graph database and Badger key-value store a contention-free cache under heavy concurrency[^1]. It is not a distributed cache and not a Redis replacement: it is a `map[K]V` with an admission and eviction brain, living inside your process. The design goal was a cache that keeps a high hit ratio while many goroutines hammer it, without the mutex contention that a naive `sync.Map` + LRU implementation suffers.

Its defining tradeoff is honesty about weakening guarantees to win throughput. Writes are batched through ring buffers and applied with eventual consistency, and a `Set` for a *new* key is explicitly allowed to be dropped — either at the write buffer or by the TinyLFU admission policy deciding the key is not worth keeping[^2]. Updates to existing keys are guaranteed, but "I called Set, therefore Get returns it" is not true immediately, or sometimes at all. That is a deliberate contract, and it is the single most common source of surprise for new users who treat it like a plain map.

The v2 line (2024) reworked the public API around Go generics, so a cache is now typed `Config[K, V]` instead of the `interface{}` v1 API. v1 and v2 are separate import paths, and mixing them or upgrading naively is a real migration step, not a drop-in bump[^3]. The repository moved under the Hypermode organization's orbit as Dgraph's commercial steward changed, but development continues on the `dgraph-io/ristretto` repo with recent commit activity.

## Getting Started

```sh
# v2 (generics) — recommended for new code
go get github.com/dgraph-io/ristretto/v2
```

```go
package main

import (
	"fmt"

	"github.com/dgraph-io/ristretto/v2"
)

func main() {
	cache, err := ristretto.NewCache(&ristretto.Config[string, string]{
		NumCounters: 1e7,     // track frequency of ~10M keys (10x expected items)
		MaxCost:     1 << 30, // 1GB total budget, measured in "cost" units
		BufferItems: 64,      // keys buffered per Get; 64 is the tuned default
	})
	if err != nil {
		panic(err)
	}
	defer cache.Close()

	cache.Set("key", "value", 1) // cost of 1
	cache.Wait()                 // block until the write buffer drains

	if v, ok := cache.Get("key"); ok {
		fmt.Println(v)
	}
	cache.Del("key")
}
```

The `cache.Wait()` call is the critical detail: without it, an immediate `Get` after `Set` often misses because the write is still in the buffer. In production you do not call `Wait` per-write (it defeats the point) — you accept eventual visibility.

## Architecture / How It Works

Ristretto splits the two hard problems of caching — *what to admit* and *what to evict* — and solves them with sampling rather than exact bookkeeping.

- **Admission: TinyLFU.** Before a new item is stored, its estimated frequency is compared against the victim it would displace; the incoming key is only admitted if it is "more popular" than what would be evicted[^4]. Frequency is tracked in a Count-Min Sketch (roughly 12 bits per counter, sized by `NumCounters`) with periodic halving so old popularity decays. This is why `NumCounters` should be set to about 10x the number of items you expect to hold, not to your item count.
- **Eviction: SampledLFU.** Rather than maintaining an exact LRU/LFU ordering (which requires a lock on every access), Ristretto samples a small set of keys and evicts the least-valuable among them. This gives near-LRU hit ratios without the global lock.
- **Cost-based sizing.** `MaxCost` is a budget in arbitrary units, not an item count. Each `Set` supplies a cost, so one large 50MB entry can evict many small ones. This suits caches of heterogeneous objects (query results, decoded images) far better than a fixed item cap.
- **Contention management.** Reads are recorded into per-goroutine `BufferItems`-sized ring buffers that are flushed in batches; writes go through a separate channel-backed buffer processed by a single goroutine. Batching plus a striped/lossy access log is where the throughput comes from — the cache accepts losing some access records under contention in exchange for never blocking readers.

The coupling story is straightforward: Ristretto is a leaf library with no network, no persistence, and its main consumers are its own siblings — Badger and Dgraph. That means the hot paths are tuned for those database workloads (many small keys, Zipfian access), which is worth knowing if your workload looks very different.

## Production Notes

- **Set is not a guarantee.** New-key `Set` can be silently dropped by the buffer or the admission policy. Never use Ristretto as a source of truth or as a write-through store where a miss implies data loss. It is a cache in the strict sense: correctness must not depend on any given entry being present.
- **Read-after-write is eventual.** A `Get` immediately following a `Set` frequently misses until buffers flush. Tests that assume synchronous visibility must call `Wait()`; production code must tolerate the miss.
- **`NumCounters` mis-sizing quietly wrecks hit ratio.** Too small and the frequency sketch saturates and admits/evicts poorly; the guidance of ~10x expected items is load-bearing, not decorative.
- **`MaxCost` must reflect real object cost.** If every `Set` passes cost `1` but your objects vary wildly in size, the cache will either overshoot memory or evict badly. Cost should approximate bytes (or whatever your budget unit is).
- **Metrics are off by default** and add overhead when enabled. Turn them on to tune, but measure the cost under your concurrency before leaving them on in production.
- **v1 → v2 is a real migration.** The generics API changes signatures and the import path (`/v2`). You cannot import both transparently; plan the switch, and check transitive dependencies (some libraries still pin v1).
- **Historical eviction bugs.** Earlier versions had reported issues where memory usage under certain cost/TTL patterns did not track `MaxCost` as expected; if you are on an old v0.0.x/v1 pin, review the changelog and upgrade rather than assuming current behavior.

## When to Use / When Not

**Use when:**
- You need a fast, concurrent, in-process cache in Go and can tolerate probabilistic admission.
- Your working set exceeds memory so eviction quality (hit ratio) actually matters.
- Objects vary in size and you want cost-based rather than count-based limits.
- You are already in the Dgraph/Badger ecosystem or a similar high-QPS Zipfian workload.

**Avoid when:**
- You need strict "Set then Get returns it" semantics — use a plain guarded map or `sync.Map`.
- You need a shared cache across processes or machines — that is Redis/Memcached territory, not Ristretto.
- Your cache is tiny and low-concurrency — the admission/eviction machinery is overkill; a simple LRU is easier to reason about.
- You require persistence, TTL as a primary feature, or eviction callbacks with hard guarantees.

## Alternatives

- hashicorp/golang-lru — simple, exact LRU (and 2Q/ARC variants); use when you want predictable, guaranteed semantics over maximum throughput.
- dgraph-io/badger — persistent embedded key-value store; use when you need durability, not just an in-memory cache (and note it uses Ristretto internally).
- patrickmn/go-cache — map with expiration and simple API; use for small, low-concurrency caches where TTL is the main need.
- maypok86/otter — newer Go cache using the S3-FIFO / W-TinyLFU family; use when you want comparable hit-ratio focus with a more modern API and active benchmarking.
- redis/go-redis — client for Redis; use when the cache must be shared across processes or survive restarts.

## History

| Version | Date | Notes |
|---------|------|-------|
| Blog announcement | 2019-08 | Introduced as a contention-free cache for Dgraph/Badger; TinyLFU + SampledLFU design[^1]. |
| v0.0.x → v0.1.x | 2019–2020 | Early API stabilization; cost-based eviction, metrics, Count-Min sketch tuning. |
| v0.1.1 | ~2022 | Widely-pinned v1-era release across the Go ecosystem. |
| v2.0.0 | 2024 | Generics-based `Config[K, V]` API on the `/v2` import path[^3]. |

(Exact tag dates vary; consult the repository's releases page for the authoritative changelog before pinning.)

## References

[^1]: Hypermode (formerly Dgraph) blog, "Introducing Ristretto: A High-Performance Go Cache." https://hypermode.com/blog/introducing-ristretto-high-perf-go-cache
[^2]: Ristretto README, FAQ — "What shortcuts are you taking?" (dropped Set calls, eventual consistency). https://github.com/dgraph-io/ristretto#faq
[^3]: Ristretto README, "Choosing a version" — v1 vs v2 generics interface. https://github.com/dgraph-io/ristretto#choosing-a-version
[^4]: Gil Einziger, Roy Friedman, Ben Manes, "TinyLFU: A Highly Efficient Cache Admission Policy" — arXiv:1512.00727. https://arxiv.org/abs/1512.00727

## Tags

go, golang, cache, in-memory-cache, tinylfu, lfu, eviction-policy, concurrency, performance, library, database
