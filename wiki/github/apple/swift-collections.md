# apple/swift-collections

> Production-grade data structures for Swift that the standard library doesn't ship — Deque, OrderedSet/OrderedDictionary, Heap, BitSet, and persistent hashed collections.

[GitHub repo](https://github.com/apple/swift-collections) ·
[Documentation](https://swiftpackageindex.com/apple/swift-collections/documentation) ·
[License: Apache-2.0](https://github.com/apple/swift-collections/blob/main/LICENSE.txt)

## Overview

Swift Collections is Apple's official package of data structure implementations that fill gaps in the Swift standard library[^1]. The stdlib ships `Array`, `Set`, and `Dictionary` and stops there; anything else — a double-ended queue, an insertion-ordered dictionary, a priority queue, a bit set — you either import this package or hand-roll. It is maintained by the same Swift core team members who own the stdlib collection types, and several of its modules are staging grounds for constructs intended to eventually migrate into the stdlib itself.

The package is deliberately conservative about its stability promise. Public API changes only in major versions per Semantic Versioning, and the stable surface is narrowly scoped: `Collections`, `DequeModule`, `OrderedCollections`, `BitCollections`, `HeapModule`, `HashTreeCollections`, plus the newer `BasicContainers` and `TrailingElementsModule`[^2]. Everything genuinely experimental is walled off behind package traits (`UnstableContainersPreview`, `UnstableHashedContainers`, `UnstableSortedCollections`) that are disabled by default and can break in any release, including patch releases.

The defining tension is that this is stdlib-adjacent code held to stdlib standards — value semantics with copy-on-write, `Sendable`/`Codable`/`Hashable` conformances, documented complexity — but shipped as a versioned package whose minimum Swift toolchain floor rises over time. Any minor release may require a newer compiler, so pinning a version is a coupling decision, not a formality.

## Getting Started

Add the package dependency in `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-collections.git", from: "1.6.0"),
],
targets: [
    .target(name: "MyApp", dependencies: [
        .product(name: "Collections", package: "swift-collections"),
    ]),
]
```

The umbrella `Collections` module re-exports the common types with a single import:

```swift
import Collections

var queue: Deque<Int> = [1, 2, 3]
queue.prepend(0)              // O(1) at both ends
queue.append(4)

var seen: OrderedSet<String> = []
seen.append("b"); seen.append("a"); seen.append("b")
// seen == ["b", "a"] — insertion order preserved, duplicate ignored

var heap: Heap<Int> = [5, 1, 3]
heap.insert(2)
heap.popMin()                 // 1  (min-max heap: popMin and popMax both O(log n))
```

To keep binary size and compile time down, import only the module you need (`import DequeModule`) rather than the umbrella.

## Architecture / How It Works

The package is a set of independent modules, each self-contained so clients pull in only what they use. The important internals differ per structure:

- **`Deque`** (`DequeModule`) is a growable ring buffer. Both ends are amortized O(1); random access is O(1). Unlike `Array`, prepending does not shift elements. It is value-typed with copy-on-write, so a copy is cheap until one side mutates.
- **`OrderedSet` / `OrderedDictionary`** (`OrderedCollections`) keep a hash table for O(1) lookup alongside a contiguous array that records insertion order. `OrderedDictionary` is not a drop-in `Dictionary`: subscripting a missing key returns `nil` but the type exposes an array-like index space, and its `Sequence` order is insertion order, not hash order.
- **`Heap`** (`HeapModule`) is a min-max heap in a flat array, giving O(1) access to both minimum and maximum and O(log n) insert/remove at either end — the reason it beats a plain binary heap for double-ended priority-queue use.
- **`BitSet` / `BitArray`** (`BitCollections`) pack bits into machine words. `BitSet` is a far denser `Set<Int>` for non-sparse integer ranges; `BitArray` a denser `[Bool]`.
- **`TreeSet` / `TreeDictionary`** (`HashTreeCollections`) implement CHAMP (Compressed Hash-Array Mapped Prefix Trees)[^3]. These are persistent (immutable-shareable) hashed collections: mutating a shared copy duplicates only the tree path that changed, not the whole structure, making them suited to snapshotting and structural sharing where standard COW would copy the entire backing store.

More recent modules push into Swift's ownership model. `BasicContainers` provides ownership-aware `UniqueArray` (uniquely held, resizable) and `RigidArray` (fixed capacity), and `TrailingElementsModule`'s `TrailingArray` is a low-level `ManagedBuffer` variant for C-style header-plus-inline-buffer layouts. These, along with the noncopyable containers gated behind `UnstableHashedContainers` (Robin Hood hashing, noncopyable elements/keys/values), are the package's role as a proving ground for stdlib primitives that depend on unreleased language features.

## Production Notes

**Not part of the Swift standard library — it is a dependency.** Adding it introduces a package to your dependency graph and its version floor. Minor releases can raise the minimum required Swift toolchain; only patch releases promise not to. If you support older toolchains, pin conservatively and read release notes before bumping a minor version.

**The stable/unstable line is load-bearing.** Everything behind `UnstableContainersPreview`, `UnstableHashedContainers`, and `UnstableSortedCollections` is not public API and can change or vanish in any release. `SortedCollections` (B-tree `SortedSet`/`SortedDictionary`) and `_RopeModule` are explicitly not stable. Do not build production code on trait-gated types unless you accept churn on every update. The absence of a stable sorted/B-tree collection is the most common thing users expect to find here and do not.

**`OrderedDictionary` is not a `Dictionary`.** It does not conform to the same protocols in every respect, iterates in insertion order, and its removal semantics (`removeValue` vs. index removal) differ. Code that assumes `Dictionary` substitutability will need review. Similarly `OrderedSet` preserves order and therefore its `==` and hashing differ from `Set`.

**Persistent collections are a specialized tool.** `TreeDictionary`/`TreeSet` win when you keep many structurally-shared snapshots or mutate copies frequently; for a single mutable collection with no sharing, standard `Set`/`Dictionary` are typically faster and lighter. Reach for CHAMP for the sharing pattern, not as a default.

**Import granularly.** The umbrella `Collections` module pulls in every stable type. In size- or compile-sensitive targets, import the specific module (`import HeapModule`) to avoid dragging in the rest.

**CMake/Xcode build configs are internal.** The repo ships CMake and Xcode configurations, but the source-stability promise applies only to the Swift Package Manager build. The other configurations exist to build the private binaries bundled in Swift toolchains and may change or be removed without notice.

## When to Use / When Not

**Use when:**
- You need a data structure the stdlib omits: `Deque`, `OrderedSet`/`OrderedDictionary`, `Heap` (priority queue), `BitSet`/`BitArray`.
- You want value semantics, COW, and stdlib-quality conformances (`Codable`, `Sendable`, `Hashable`) rather than a third-party reimplementation.
- You keep many shared snapshots of a collection and can exploit CHAMP structural sharing.

**Avoid when:**
- You need a stable sorted / B-tree collection — the ones here are trait-gated and unstable.
- You want zero dependencies and can hand-roll the one structure you need.
- You need to support an older Swift toolchain than the current minor release requires.
- Your use is a single non-shared mutable dictionary/set — the stdlib types are usually the better default.

## Alternatives

- raywenderlich/swift-algorithm-club — teaching-oriented reference implementations; read to learn, not a maintained dependency.
- apple/swift-algorithms — sibling Apple package for lazy sequence/collection algorithms (chunked, windows, combinations) rather than container types; complementary, not competing.
- apple/swift-async-algorithms — use when you need `AsyncSequence` operators instead of concrete data structures.
- pointfreeco/swift-identified-collections — use when you specifically want an ID-keyed ordered collection for app state; it is built on this package's `OrderedDictionary`.
- Hand-rolled structures — use when you need exactly one structure and want to avoid the dependency and its rising toolchain floor.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2021-04-05 | First public preview: Deque, OrderedSet, OrderedDictionary[^1]. |
| 1.0.0 | 2021-09-10 | First source-stable release. |
| 1.1.0 | 2024-02-08 | Added BitCollections, HeapModule, and HashTreeCollections (CHAMP persistent collections)[^3]. |
| 1.2.0 | 2025-05-19 | Minor feature release. |
| 1.3.0 | 2025-09-29 | Minor feature release. |
| 1.4.0 | 2026-03-07 | Ownership-aware container work (BasicContainers). |
| 1.5.0 | 2026-05-08 | Public API defined across expanded module set[^2]. |
| 1.6.0 | 2026-06-08 | Latest release. |

## References

[^1]: Swift.org blog, "Swift Collections" announcement — 2021-04-05. https://www.swift.org/blog/swift-collections/
[^2]: swift-collections README, "Project Status" and "Definition of Public API". https://github.com/apple/swift-collections/blob/main/README.md
[^3]: Michael J. Steindorfer and Jurgen J. Vinju, "Optimizing Hash-Array Mapped Tries for Fast and Lean Immutable JVM Collections" (CHAMP), OOPSLA 2015. https://michael.steindorfer.name/publications/oopsla15.pdf

## Tags

swift, data-structures, collections, deque, ordered-dictionary, priority-queue, bitset, persistent-data-structures, apple, swift-package-manager
