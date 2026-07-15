# samber/lo

> A Lodash-style utility library for Go, built on 1.18+ generics — Map, Filter, Reduce, and ~350 other slice/map/channel helpers.

[GitHub repo](https://github.com/samber/lo) ·
[Official website](https://lo.samber.dev) ·
[License: MIT](https://github.com/samber/lo/blob/master/LICENSE)

## Overview

`lo` is a generic functional-utility library for Go, released by Samuel Berthe within days of Go 1.18 shipping generics in March 2022[^1]. It fills the gap left by Go's deliberately minimal standard library: `Map`, `Filter`, `Reduce`, `GroupBy`, `Uniq`, `Chunk`, set operations, tuple/zip helpers, and a long tail of conveniences that JavaScript's Lodash or Python's itertools provide but Go historically forced you to hand-roll with `for` loops. As of 2026 it is the most-starred functional helper library in the Go ecosystem, and one of the earliest large libraries to bet the whole API surface on generics.

The library's defining tension is with Go's own culture and, increasingly, its standard library. Idiomatic Go favors explicit loops over abstraction, and since Go 1.21 the standard `slices` and `maps` packages cover a growing slice of `lo`'s surface (`Contains`, `IndexOf`, `Sort`, `Keys`, `Values`, `Equal`). The author acknowledges this overlap directly in the README and argues the remaining abstractions still earn their place[^2]. Whether importing `lo` is worth it over a three-line loop is a recurring debate in Go code review; the honest answer is that it buys readability at the cost of allocation and a dependency, and the trade shifts every time the stdlib absorbs another helper.

`lo` has no dependencies outside the Go standard library, follows SemVer strictly, and promises no breaking changes to exported APIs before a hypothetical v2.0.0 — experimental code is quarantined under `exp/`[^2].

## Getting Started

```sh
go get github.com/samber/lo@v1
```

```go
package main

import (
    "fmt"
    "strconv"

    "github.com/samber/lo"
)

func main() {
    nums := []int{1, 2, 3, 4, 5, 6}

    even := lo.Filter(nums, func(x, _ int) bool { return x%2 == 0 })
    // []int{2, 4, 6}

    strs := lo.Map(even, func(x, _ int) string { return strconv.Itoa(x) })
    // []string{"2", "4", "6"}

    groups := lo.GroupBy(nums, func(x int) int { return x % 3 })
    // map[int][]int{0: {3, 6}, 1: {1, 4}, 2: {2, 5}}

    fmt.Println(strs, groups)
}
```

The predicate signatures pass an index as the second argument (`func(item T, index int)`), mirroring Lodash — a common surprise for first-time users who write `func(x int)` and get a compile error.

## Architecture / How It Works

`lo` is a flat collection of generic top-level functions in a single package, plus a handful of sub-packages that change evaluation semantics:

- **`lo`** (core) — eager, allocating, immutable-by-default. Every helper returns a freshly allocated result and never mutates its input.
- **`lo/parallel` (`lop`)** — the same helpers, but each element's callback runs in its own goroutine. Results are collected back in order.
- **`lo/mutable` (`lom`)** — in-place variants that modify the input slice, trading safety for zero allocation.
- **`lo/it` (`loi`)** — iterator-based lazy variants built on Go 1.23's range-over-func (`iter.Seq`), which let you compose operations without materializing intermediate slices.

The core design is eager and non-lazy: `lo.Map(lo.Filter(xs, ...), ...)` allocates one intermediate slice for the filter result and a second for the map result. There is no fluent chain object and no deferred execution in the core package — each call is a standalone function that fully evaluates before the next. This keeps the mental model simple and the stack traces readable, but it means pipelines allocate proportionally to their length. The `it` package exists specifically to address this for hot paths.

Generics are resolved by Go's monomorphization: each concrete type instantiation of `lo.Map[int, string]` generates specialized code at compile time, so there is no runtime reflection (unlike the pre-generics `thoas/go-funk`). Type inference usually works, but functions with multiple type parameters — `Map[T, R]`, `Reduce[T, R]` — occasionally need explicit type arguments when the compiler can't infer the return type from the callback alone.

Beyond collections, the library ships conditional helpers (`Ternary`, `If/ElseIf/Else`, `Switch/Case`), error-handling wrappers (`Must`, `Try`, `TryCatch`), tuple types (`T2`–`T9`, `Zip`, `Unzip`), and concurrency utilities (`Debounce`, `Throttle`, `Attempt`, `Async`, `WaitFor`). Most collection helpers now have an `...Err` sibling (`FilterErr`, `MapErr`, `ReduceErr`) that lets the callback return an error and short-circuits the operation.

## Production Notes

**`lo/parallel` spawns one goroutine per element with no concurrency limit.** `lop.Map` over a million-element slice launches a million goroutines simultaneously. For CPU-bound work this is counterproductive (scheduler thrash) and for I/O-bound work it can exhaust file descriptors or overwhelm a downstream service. Use a bounded worker pool (`sourcegraph/conc`, `golang.org/x/sync/errgroup` with `SetLimit`) when the input size is large or unbounded.

**`Ternary` evaluates both branches.** Go has no lazy conditional expression, so `lo.Ternary(cond, expensive(), fallback())` calls *both* `expensive()` and `fallback()` before selecting. Worse, `lo.Ternary(x != nil, x.Field, defaultVal)` panics on a nil `x` because both arguments are evaluated eagerly. Use `lo.TernaryF(cond, func() T {...}, func() T {...})` — which defers each branch behind a closure — whenever a branch is expensive or nil-guarded.

**The `Must*` family panics.** `lo.Must(v, err)` returns `v` and panics if `err != nil`. This is convenient in `init()`, tests, and program startup, but panicking on an expected error path is un-idiomatic and dangerous inside request handlers or long-running services. Reserve it for genuinely unrecoverable-at-startup conditions.

**Allocation cost is real on hot paths.** Because the core is eager and immutable, a chained pipeline allocates an intermediate slice per stage. In a benchmark-sensitive inner loop, a hand-written single-pass `for` loop is both faster and allocation-free. Reach for `lo/mutable` (in-place) or `lo/it` (lazy iterators) when profiling points at these allocations, or drop to a manual loop.

**Standard-library overlap keeps growing.** On Go 1.21+, prefer `slices.Contains`, `slices.Index`, `slices.Sort`, `maps.Keys`, and `maps.Values` over their `lo` equivalents where the behavior matches — they are maintained by the Go team and carry no third-party dependency. `lo` remains valuable for the helpers the stdlib does not provide (`GroupBy`, `Chunk`, `Partition`, `Uniq`, tuples), but blanket-importing it for `Contains` alone is hard to justify in 2026.

**Map-derived helpers return unordered results.** `lo.Keys`, `lo.Values`, and `lo.Entries` iterate Go maps, whose iteration order is randomized by the runtime. Do not rely on ordering; sort explicitly if you need determinism.

## When to Use / When Not

**Use when:**
- You want readable, declarative transforms (`GroupBy`, `Chunk`, `Partition`, `Uniq`, `KeyBy`) that the stdlib doesn't provide.
- Your codebase already leans functional and the team values consistency over micro-optimizing every loop.
- You need tuple/zip helpers, set operations, or the conditional/error wrappers as a cohesive, zero-dependency package.

**Avoid when:**
- You only need `Contains`/`IndexOf`/`Sort`/`Keys` — the stdlib `slices` and `maps` packages cover those without a dependency.
- You're in an allocation- or latency-critical hot path where an explicit single-pass loop wins.
- Your team writes idiomatic "boring" Go and treats functional abstraction as a review smell.
- You need lazy streaming over infinite sequences — reach for the `it` package, or the author's `samber/ro` for reactive/event-driven streams.

## Alternatives

- samber/mo — same author; monadic `Option`, `Result`, `Either`, `Future` types. Use instead of (or alongside) `lo` when you want typed absence/error handling rather than collection helpers.
- Go stdlib `slices` + `maps` — use instead when you only need the common operations the standard library already provides (Go 1.21+), to avoid a dependency.
- thoas/go-funk — the pre-generics predecessor built on reflection. Use only on Go < 1.18; it is slower and not type-safe.
- elliotchance/pie — generics-based, offers a chainable fluent API (`pie.Of(xs).Filter(...).Map(...)`). Use instead when you prefer method chaining over free functions.
- ahmetb/go-linq — LINQ-style lazy query API. Use instead when you want deferred, streaming query composition closer to C#'s LINQ.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2022-03-02 | Initial release, days after Go 1.18 generics shipped[^1]. |
| v1.11.0 | 2022-03-20 | Rapid early expansion of the helper set. |
| v1.38.0 | 2023-03-21 | Continued growth; parallel/mutable sub-package split matured. |
| v1.39.0 | 2023-12-02 | Broadened error-returning (`...Err`) variants. |
| v1.44.0 | 2024-06-30 | More slice/map coverage as stdlib overlap grew. |
| v1.47.0 | 2024-08-13 | Ongoing helper additions. |
| v1.50.0 | 2025-04-26 | 50th minor; `it` lazy-iterator package on Go 1.23 range-over-func. |
| v1.53.0 | 2026-03-02 | Latest release; still v1, no breaking API changes to date[^2]. |

The version cadence has been unusually fast — over 50 minor releases and zero major bumps in four years — reflecting the strict "additive within v1" policy. As of this writing the repo carries roughly 21.4k stars and 940 forks, and is actively maintained (last push July 2026)[^3].

## References

[^1]: Go 1.18 release notes — generics landed 2022-03-15; `samber/lo` v1.0.0 tagged 2022-03-02. https://go.dev/doc/go1.18
[^2]: `samber/lo` README — versioning policy, stdlib-overlap discussion, sub-package layout. https://github.com/samber/lo
[^3]: GitHub REST API, `repos/samber/lo` — stars, forks, last-push metadata (fetched 2026-07-15). https://api.github.com/repos/samber/lo

## Tags

go, golang, generics, functional-programming, lodash, utility-library, slices, maps, collections, higher-order-functions, zero-dependency
