# apple/swift-algorithms

> Commonly used sequence and collection algorithms for Swift — a curated, source-stable superset of what the standard library ships.

[GitHub repo](https://github.com/apple/swift-algorithms) ·
[Documentation](https://swiftpackageindex.com/apple/swift-algorithms/documentation/algorithms) ·
[License: Apache-2.0](https://github.com/apple/swift-algorithms/blob/main/LICENSE.txt)

## Overview

Swift Algorithms is an Apple-maintained SwiftPM package that adds sequence and
collection operations the standard library does not (yet) provide: chunking,
windowing, combinations, permutations, `uniqued`, `adjacentPairs`, sampling,
striding, and related lazy wrapper types. It was announced on swift.org in
October 2020 alongside swift-collections and swift-numerics as part of Apple's
push to develop parts of the standard library in the open[^1].

The package occupies a specific niche: a staging ground for functionality that
may eventually be proposed for the standard library, plus a stable home for
operations too specialized to ever merge. Its closest analogue is Python's
`itertools` (which the project's topics list directly). Everything is a `public`
extension on `Sequence`/`Collection` or a small lazy wrapper type; there is no
runtime, no global state, and nothing to configure.

The defining tension is dependency-versus-hand-roll. Every algorithm here is
something a competent Swift developer could write in a few lines, so the package
buys you tested, documented, edge-case-correct implementations at the cost of a
third-party dependency for what looks like trivial code. For library authors
that is a real consideration; for applications it is close to free.

## Getting Started

Add it to `Package.swift`:

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/apple/swift-algorithms", from: "1.2.0"),
],
targets: [
    .target(name: "MyTarget", dependencies: [
        .product(name: "Algorithms", package: "swift-algorithms"),
    ]),
]
```

```swift
import Algorithms

// Break a collection into ascending runs.
let numbers = [10, 20, 30, 10, 40, 40, 10, 20]
let runs = numbers.chunked(by: { $0 <= $1 })
// [[10, 20, 30], [10, 40, 40], [10, 20]]

// Sliding windows, adjacent pairs, and deduplication.
Array([1, 2, 3, 4].windows(ofCount: 2))   // [[1, 2], [2, 3], [3, 4]]
Array([1, 2, 3].adjacentPairs())           // [(1, 2), (2, 3)]
Array([1, 1, 2, 3, 3].uniqued())           // [1, 2, 3]

// Combinatorics — note these grow fast.
Array([1, 2, 3].combinations(ofCount: 2))  // [[1, 2], [1, 3], [2, 3]]
```

## Architecture / How It Works

The library is protocol extensions plus a handful of custom collection types.
An eager method like `chunked(by:)` returns a materialized `[SubSequence]`; the
lazy variant on a `LazySequence` returns a wrapper (`ChunkedByCollection`) that
computes elements on demand and forwards `Collection` conformance, so it
composes with other lazy operations without intermediate allocations. This
eager/lazy split mirrors the standard library's own `lazy` design and is the
single most important thing to understand about performance here.

Notable families:

- **Chunking** — `chunked(by:)` splits on adjacent-element predicates,
  `chunked(on:)` when a projected value changes, `chunks(ofCount:)` into
  fixed-size slices.
- **Windowing / striding** — `windows(ofCount:)` yields overlapping slices;
  `striding(by:)` yields every nth element.
- **Combinatorics** — `combinations(ofCount:)`, `permutations(ofCount:)`,
  `product(_:_:)`. Output size is combinatorial in the input.
- **Selection** — `min(count:)`/`max(count:)` (partial sort), `randomSample`/
  `randomStableSample` (reservoir sampling), `trimmingPrefix`/`suffix`.
- **Partitioning & indexing** — `stablePartition`, `partitioningIndex`,
  `indexed()`, `grouped(by:)`, `keyed(by:)`.

The `Guides/` directory in the repo documents the design rationale and
complexity of each operation as a set of proposal-style markdown files, which
is the authoritative reference for semantics and Big-O guarantees. The package
carries the Apache-2.0 license with the Swift runtime library exception and
commits to source stability under Semantic Versioning: source-breaking changes
to public API only land in a new major version[^2].

## Production Notes

**Eager methods allocate.** Any method returning an `Array` or `[SubSequence]`
materializes the whole result. For large inputs, reach for the `.lazy` variant
or a manual loop; profiling frequently shows an eager `chunked`/`windows` on a
big array as an unexpected allocation hotspot.

**Combinatorial explosion is real.** `combinations(ofCount: k)` on n elements
produces C(n, k) results and `permutations` are worse. These are genuinely easy
to misuse — a `combinations(ofCount: 3)` over a few hundred items is millions of
allocations. There is no built-in guard; the caller owns the cardinality.

**Minor version bumps can raise the Swift toolchain floor.** The maintainers
state explicitly that adopting new Swift language/toolchain features may require
clients to move to a newer compiler, and that such a requirement only needs a
*minor* version bump[^2]. If you pin with `from:` and expect only additive
changes, a routine `1.x` update can still demand a newer Xcode/Swift than your
CI has. Pin ranges accordingly.

**Only non-underscored public API is stable.** Anything underscored or SPI can
change in any release, including patch releases. Don't reach into internals.

**Small footprint, but still a dependency.** The module compiles into your
binary with no dynamic library and no runtime overhead beyond the code you call.
The cost is dependency-graph surface, not size or speed — and library authors
who expose it transitively pass the toolchain-floor policy above to consumers.

## When to Use / When Not

**Use when:**
- You reach for `itertools`-style operations (chunking, windows, combinations,
  sampling) and want tested, documented implementations instead of ad-hoc ones.
- You value edge-case correctness (empty collections, single elements, overflow)
  over shaving a dependency.
- You are already in the Apple SwiftPM ecosystem and want first-party stability
  guarantees.

**Avoid when:**
- You need one trivial helper (`adjacentPairs` alone) and adding a dependency is
  not worth it — copy the few lines instead.
- You are working over `AsyncSequence`; use swift-async-algorithms.
- You need data structures (deque, ordered set, heap) rather than algorithms;
  that is swift-collections' job.

## Alternatives

- apple/swift-collections — use instead when you need data structures (Deque,
  OrderedSet, OrderedDictionary, Heap) rather than sequence operations; the two
  are complementary, not competing.
- apple/swift-async-algorithms — use instead when your operations run over
  `AsyncSequence` (debounce, merge, chunked-by-time) rather than synchronous
  collections.
- apple/swift-numerics — use instead for numeric protocols, complex numbers,
  and elementary functions; different domain, same maintainer family.
- The Swift standard library — use instead when you only need map/filter/reduce,
  sorting, and `prefix`/`suffix`; don't add a package for what's already built in.
- raywenderlich/swift-algorithm-club — a learning resource of algorithm
  implementations, not a versioned dependency; read it, don't `import` it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2020-10 | Initial open-source release with the swift.org announcement[^1]. |
| 1.0.0 | 2021 | First source-stable major release under SemVer[^2]. |
| 1.1.0 | 2022 | Additional operations and refinements (additive minor). |
| 1.2.0 | 2024 | Current release; the version the README pins as `from:`[^3]. |

## References

[^1]: Swift.org, "Swift Algorithms" announcement — October 2020. https://swift.org/blog/swift-algorithms/
[^2]: Swift Algorithms README, "Source Stability" section (Apache-2.0 + SemVer; minor bumps may raise the toolchain floor). https://github.com/apple/swift-algorithms#source-stability
[^3]: Swift Algorithms README, "Adding Swift Algorithms as a Dependency" (`from: "1.2.0"`). https://github.com/apple/swift-algorithms#adding-swift-algorithms-as-a-dependency
[^4]: API documentation, Swift Package Index. https://swiftpackageindex.com/apple/swift-algorithms/documentation/algorithms

## Tags

swift, algorithms, sequence, collection, itertools, functional, swiftpm, apple, standard-library, lazy-evaluation
