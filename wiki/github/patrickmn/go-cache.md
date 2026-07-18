# patrickmn/go-cache

> A thread-safe in-memory key:value cache with expiration for Go — a
> `map[string]interface{}` with TTLs, and one of Go's most-depended-on
> libraries despite being effectively frozen since 2017.

[GitHub repo](https://github.com/patrickmn/go-cache) ·
[Official website](https://patrickmn.com/projects/go-cache/) ·
[License: MIT](https://github.com/patrickmn/go-cache/blob/master/LICENSE)

## Overview

go-cache is a single-machine, in-process cache library for Go, started by
Patrick Mylund Nielsen in January 2012[^1]. The README's own framing is
accurate and modest: it is essentially a thread-safe `map[string]interface{}`
with per-item expiration times[^2]. Because it lives inside your process, there
is no serialization, no network hop, and no separate daemon — the trade being
that the cache dies with the process and cannot be shared across machines.

At ~8.8k stars and ~900 forks it remains one of the most-referenced Go caching
libraries, but the numbers deserve interpretation: the last tagged release is
v2.1.0 from 2017, the last commit to master landed in November 2023, and 76
issues/PRs sit open without triage. It is abandonware in practice — stable
abandonware, since the code is small (~1,200 lines) and the problem it solves
does not move. Many teams vendor it or copy the pattern rather than depend on
it. The defining tension: it is the simplest possible correct answer to "I need
a TTL map", and simultaneously the wrong answer for any workload with bounded
memory requirements, high write concurrency, or millions of entries.

A modules-era footgun follows from its age: the repo tagged `v2.x` before Go
modules existed and never added a `/v2` module path, so `go get` resolves it as
`v2.1.0+incompatible`[^3].

## Getting Started

```bash
go get github.com/patrickmn/go-cache   # resolves to v2.1.0+incompatible
```

```go
package main

import (
	"fmt"
	"time"

	"github.com/patrickmn/go-cache"
)

func main() {
	// Default TTL 5 min; expired items purged every 10 min.
	c := cache.New(5*time.Minute, 10*time.Minute)

	c.Set("foo", "bar", cache.DefaultExpiration)
	c.Set("meaning", 42, cache.NoExpiration) // lives until deleted

	if x, found := c.Get("foo"); found {
		s := x.(string) // values are interface{}; assertion required
		fmt.Println(s)
	}
}
```

The library predates Go generics (1.18), so every read requires a type
assertion. The README's own advice for performance is "store pointers"[^2] —
storing large structs by value costs a copy plus an allocation per `Set`.

## Architecture / How It Works

The entire library is one package with a handful of files. The core is:

- **One map, one lock.** `map[string]Item` guarded by a single
  `sync.RWMutex`. `Item` is `{Object interface{}; Expiration int64}`, with
  expiration stored as a Unix-nanosecond timestamp; `0` means no expiration.
  Reads take the read lock; every `Set`, `Delete`, and `Increment` takes the
  write lock. There is no sharding in the public API (an unexported sharded
  variant exists in the repo but was never stabilized).
- **Lazy + background expiration.** `Get` compares `time.Now().UnixNano()`
  against the item's expiration, so expired items are never returned. Actual
  memory reclamation happens in a "janitor" goroutine that calls
  `DeleteExpired()` on the cleanup interval passed to `New`.
- **The finalizer trick.** The janitor goroutine would normally pin the cache
  and leak. `New` returns an outer `*Cache` wrapping an inner `cache` struct;
  the janitor references only the inner struct, and `runtime.SetFinalizer` on
  the outer wrapper stops the janitor when the last user reference is
  dropped[^4]. This is the one genuinely clever piece of the codebase and a
  frequently-cited example of finalizer use in Go.
- **Typed increments.** `Increment`/`Decrement` plus a family of
  `IncrementInt`, `IncrementFloat64`, etc., because `interface{}` arithmetic
  requires a type switch per numeric type.
- **Optional persistence.** `Items()` + `NewFrom()` (and deprecated
  `SaveFile`/`LoadFile` using `encoding/gob`) allow snapshot/restore for
  warm restarts. The docs themselves flag the caveats: gob needs concrete
  type registration and the mechanism is not a durability story[^2].

There are no dependencies outside the standard library, which is a large part
of why the library has survived nine years without maintenance.

## Production Notes

- **No size bound, no eviction policy.** Expiration is the only way entries
  leave. There is no LRU/LFU, no max-entries, no cost accounting. A cache
  keyed by unbounded input (user IDs, URLs, query strings) grows until OOM.
  This is the most common production incident with go-cache.
- **Write-lock contention.** The single RWMutex is fine up to moderate
  concurrency, but write-heavy workloads on many cores serialize on the one
  lock. Benchmarks in the bigcache/freecache literature exist precisely
  because of this ceiling[^5].
- **GC pressure at scale.** Millions of `interface{}` values and string keys
  mean millions of pointers for the garbage collector to scan each cycle.
  At large entry counts, GC pause impact — not lock contention — is usually
  the first symptom. bigcache and freecache were designed specifically to
  sidestep this by storing serialized bytes[^5].
- **Cleanup interval is a stop-the-world-ish scan.** `DeleteExpired` walks
  the whole map under the write lock. With very large maps and short cleanup
  intervals, janitor runs cause periodic latency spikes on writers.
- **No `Get`-with-TTL-refresh, no singleflight.** Cache stampede protection
  (many goroutines missing the same key simultaneously and all recomputing)
  must be layered on with `golang.org/x/sync/singleflight`.
- **Unmaintained means unmerged.** Generics support, sharding, and eviction
  PRs have been open for years. Pin the version and treat the code as yours;
  do not expect upstream fixes.

## When to Use / When Not

**Use when:**
- You need a TTL map for a small-to-medium working set (thousands to low
  hundreds of thousands of entries) in a single process.
- Key cardinality is naturally bounded, so absence of eviction is safe.
- You want zero dependencies and code simple enough to audit in one sitting.

**Avoid when:**
- Memory must be bounded — you need LRU/cost-based eviction (ristretto,
  golang-lru, otter).
- You cache millions of entries and GC pauses matter (bigcache, freecache).
- You want type safety without assertions — post-1.18 generic caches exist.
- You need cross-process or cross-machine sharing — that is Redis/memcached
  territory, not an in-process cache.

## Alternatives

- dgraph-io/ristretto — use instead when you need bounded memory with a
  high hit ratio; TinyLFU admission and cost-based eviction.
- allegro/bigcache — use instead for millions of entries; stores serialized
  bytes to keep entries out of GC scan scope.
- coocood/freecache — same zero-GC-overhead goal as bigcache with a strict
  preallocated memory bound.
- hashicorp/golang-lru — use instead when you want a simple size-bounded LRU
  rather than TTL semantics; actively maintained.
- jellydator/ttlcache — use instead for the same TTL-map semantics with
  generics and active maintenance.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-01 | First commit; GOPATH-era library[^1]. |
| v2.0.0 | 2015 | API cleanup; `time.Duration` expirations; v2 tag predates modules. |
| v2.1.0 | 2017-06 | Last tagged release; resolves as `+incompatible` under modules[^3]. |
| — | 2023-11 | Last commit to master; no release cut. Project dormant since. |

## References

[^1]: GitHub repository metadata — created 2012-01-02, last push 2023-11-20.
    https://github.com/patrickmn/go-cache
[^2]: go-cache README. https://github.com/patrickmn/go-cache#readme
[^3]: Go Modules reference, "+incompatible versions" — v2+ tags without a
    go.mod major-version suffix. https://go.dev/ref/mod#incompatible-versions
[^4]: go-cache API documentation (janitor and `New` semantics).
    https://pkg.go.dev/github.com/patrickmn/go-cache
[^5]: allegro engineering, "Writing a very fast cache service in Go" —
    rationale for bigcache's GC-avoiding design.
    https://blog.allegro.tech/2016/03/writing-fast-cache-service-in-go.html

## Tags

go, caching, in-memory, key-value, ttl, concurrency, standard-library-only, single-machine
