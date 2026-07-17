# nlohmann/fifo_map

> A header-only C++ associative container with the `std::map` interface that orders elements by insertion time instead of by key.

[GitHub repo](https://github.com/nlohmann/fifo_map) ·
[License: MIT](https://github.com/nlohmann/fifo_map/blob/master/LICENSE.MIT)

## Overview

`fifo_map` is a single-header C++ container from Niels Lohmann that mimics
`std::map` but keeps entries in **first-in-first-out (insertion) order** rather
than sorted by key. It exposes the same interface as `std::map`, so it can be
used as a drop-in replacement wherever iteration order should follow the order
keys were added[^1]. The entire implementation lives in one file,
`src/fifo_map.hpp`, and depends only on the STL.

The project's origin explains its shape: it was extracted from
[nlohmann/json](https://github.com/nlohmann/json) to give that library a way to
preserve object key order[^2]. It is small, single-purpose, and effectively
feature-frozen — the first tagged release, `v1.0.0`, did not appear until 2020,
five years after the code was written, and `v1.0.1` (2025) was a packaging
bump, not new functionality[^3].

The defining tension is that **the author now steers new users elsewhere.** The
README ends with a pointer to `ordered_map`, a lighter insertion-ordered
container that lives inside nlohmann/json and receives active maintenance[^4].
`fifo_map` remains correct and usable, but it is best understood as a stable,
minimal utility rather than an evolving project. Treat its ~200 stars as a
signal of a well-scoped micro-library, not a growing ecosystem.

## Getting Started

There is no package to install — vendor the single header (or add it via
CMake `FetchContent` / a submodule) and include it:

```cpp
#include "fifo_map.hpp"
#include <iostream>
#include <string>

using nlohmann::fifo_map;

int main() {
    fifo_map<int, std::string> m;
    m[2] = "two";
    m[3] = "three";
    m[1] = "one";

    for (const auto& [key, val] : m)   // prints 2, 3, 1 — insertion order
        std::cout << key << ": " << val << "\n";

    m.erase(2);
    m[2] = "zwei";                     // re-inserted key moves to the END

    for (const auto& [key, val] : m)   // prints 3, 1, 2
        std::cout << key << ": " << val << "\n";
}
```

Compiles with any C++11 (or newer) toolchain; no build step, no linkage. The
bundled `make && ./unit` target runs the Catch-based test suite.

## Architecture / How It Works

`fifo_map` is not a new data structure — it is `std::map` with a custom
comparator that sorts by insertion index instead of by key value. Three pieces
cooperate:

1. A side table, `std::unordered_map<Key, std::size_t>`, mapping each key to a
   **monotonically increasing** insertion counter. This is the source of truth
   for order.
2. A comparator, `fifo_map_compare`, that holds a pointer to that side table and
   answers "does key A come before key B?" by comparing their stored indices.
3. The underlying `std::map<Key, T, fifo_map_compare>`, whose red-black tree is
   therefore ordered by insertion index rather than by key.

Because the comparator is stateful and references external memory, the container
carries the side table as a member and wires the comparator's pointer to it.
Insertion assigns the next counter value; iteration walks the tree in index
order. This is why a `Key` must be usable both by `std::map` (needs ordering for
the tree, provided indirectly) and by `std::unordered_map` (needs `std::hash`
and equality) — a subtle extra requirement over plain `std::map`.

Complexity: `operator[]`, `insert`, and `erase` add the cost of an
`unordered_map` insert/erase — O(1) average, O(n) worst case — on top of the
`std::map` tree operation. All read-only operations match `std::map`
performance[^1]. Each key is stored twice: once in the tree node and once in the
index table.

## Production Notes

- **Re-inserting a key moves it to the back.** The insertion counter only ever
  increases, so `erase(k)` followed by `m[k] = v` gives `k` a new, higher index
  and places it last in iteration order. This is intentional FIFO semantics but
  surprises anyone expecting a key to keep its original slot. There is no
  "update in place without reordering" mode.
- **The counter never resets.** It is a `std::size_t` that increments on every
  insertion for the lifetime of the object. Practically unbounded, but the
  ordering is index-based, not timestamp-based — long-lived containers with
  heavy churn accumulate large indices even after erases.
- **Stateful comparator = copy/move care.** The comparator holds a pointer into
  the container's own side table. Copying or moving a `fifo_map` must re-point
  that comparator at the new object's table; this is handled internally, but it
  makes the type heavier to reason about than a stateless-comparator `std::map`,
  and it is the kind of internal invariant worth trusting only the tested code
  path for.
- **Not thread-safe.** Like all standard containers, concurrent writes need
  external synchronization; there is nothing FIFO-specific that helps here.
- **No versioned dependency story.** Distribution is vendor-the-header. Pin to a
  commit or the `v1.0.1` tag; do not assume a package-manager entry. Header
  guards use the `nlohmann` namespace, which can collide conceptually with
  nlohmann/json's own headers if you mix both — they are separate projects with
  overlapping names.
- **Maintenance is minimal by design.** Three open issues, sparse commit
  cadence, and an author-endorsed successor mean you should not expect new
  features or fast responses. For a frozen 200-line utility this is acceptable;
  just size your expectations accordingly.

## When to Use / When Not

**Use when:**
- You need `std::map`'s full interface but want iteration in insertion order,
  and you want a zero-dependency, single-header drop-in.
- You are already reading key order out of parsed data (config, JSON-like) and
  round-tripping it must preserve order.
- You want something small and auditable you can vendor and forget.

**Avoid when:**
- You already depend on nlohmann/json — use its bundled `ordered_map` instead;
  it is the author's maintained successor.
- Lookup throughput matters most: the extra `unordered_map` and red-black tree
  make it slower and heavier than a purpose-built ordered hash map.
- You need in-place value updates that keep a key's original position — FIFO
  semantics reorder on re-insert.
- You want an actively developed dependency with releases and issue triage.

## Alternatives

- nlohmann/json — its internal `ordered_map` is the author's recommended
  replacement; use it when you already pull in that library.
- Tessil/ordered-map — a dedicated insertion-ordered hash map with faster
  lookups and more features; use it when order plus lookup speed both matter.
- Tessil/robin-map — use when you need a fast hash map and do *not* care about
  ordering.
- boostorg/multi_index — use when you need multiple simultaneous orderings
  (insertion order *and* key lookup *and* more) and can accept Boost's weight.
- For tiny maps, a plain `std::vector<std::pair<K,V>>` with linear scan often
  beats any of the above on both simplicity and cache behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2015-08-15 | Extracted to support key ordering in nlohmann/json[^2]. |
| (untagged) | 2015–2017 | Copyright range in the license header; used in the wild long before a release[^1]. |
| v1.0.0 | 2020-07-09 | First tagged release[^3]. |
| v1.0.1 | 2025-11-23 | Packaging/maintenance bump; no new features[^3]. |

## References

[^1]: nlohmann/fifo_map README — overview, complexity, and example. https://github.com/nlohmann/fifo_map
[^2]: nlohmann/json object key-order discussion, the use case fifo_map was built for. https://github.com/nlohmann/json/issues/485
[^3]: nlohmann/fifo_map releases (v1.0.0 2020-07-09, v1.0.1 2025-11-23). https://github.com/nlohmann/fifo_map/releases
[^4]: `ordered_map`, the author's recommended successor, bundled in nlohmann/json. https://github.com/nlohmann/json/blob/develop/include/nlohmann/ordered_map.hpp

## Tags

cpp, header-only, associative-container, insertion-order, fifo, stl, std-map-drop-in, single-header, data-structures, nlohmann
