# martinus/robin-hood-hashing

> Single-header C++ hash map/set that trades pointer stability for speed and memory efficiency over `std::unordered_map` — now archived in favor of a successor.

[GitHub repo](https://github.com/martinus/robin-hood-hashing) ·
[Gitter chat](https://gitter.im/martinus/robin-hood-hashing) ·
[License: MIT](https://github.com/martinus/robin-hood-hashing/blob/master/LICENSE)

## Overview

`robin_hood::unordered_map` and `robin_hood::unordered_set` are drop-in replacements for the C++ standard library's `std::unordered_map` / `std::unordered_set`, distributed as a single header (`robin_hood.h`) with no dependencies. The project targets C++11 through C++20 and is designed to be both faster and more memory-efficient than the standard containers for real-world workloads[^1]. The name comes from the Robin Hood open-addressing scheme it uses internally.

The defining tension is standard-library compatibility versus performance. `std::unordered_map` is a specification that effectively mandates node-based storage and pointer/reference stability across insertions; that guarantee costs an allocation per element and a pointer chase per lookup. robin_hood breaks from the standard where the standard is slow — its default flat layout stores values inline and does not keep references stable — while keeping a familiar API surface so most call sites compile unchanged.

As of 2026 the repository is **archived and no longer maintained**. The author has published a successor, `ankerl::unordered_dense` (martinus/unordered_dense), and states robin_hood will only receive PRs for critical bug fixes[^2]. New code should prefer the successor; robin_hood remains widely vendored in existing C++ projects and is stable for what it does, but it will not track new standards or fix non-critical issues.

## Getting Started

The simplest integration is to copy the single header into your project:

```bash
# Download the header from the latest release, or vendor it directly
curl -LO https://raw.githubusercontent.com/martinus/robin-hood-hashing/master/src/include/robin_hood.h
```

```cpp
#include "robin_hood.h"
#include <string>

int main() {
    robin_hood::unordered_map<std::string, int> counts;
    counts["apple"]++;
    counts["apple"]++;
    counts["pear"] += 3;

    for (auto const& kv : counts) {
        // note: iteration order is unspecified, like std::unordered_map
        // kv.first, kv.second
    }
    return counts["apple"]; // 2
}
```

It is also packaged for Conan as `robin-hood-hashing` (e.g. `robin-hood-hashing/3.11.5`), maintained by the community via conan-center-index[^1].

## Architecture / How It Works

Robin Hood hashing is an open-addressing scheme that bounds the variance of probe lengths by "stealing from the rich": on insertion, if the element being placed has probed farther from its ideal slot than the element currently occupying a slot, the two are swapped, so no key sits far from its home while another sits close. This keeps probe sequences short and predictable, which is what makes lookups fast.

robin_hood stores, alongside the bucket array, a compact per-slot **info byte** that packs a hash fingerprint together with the distance-from-ideal offset. Lookups compare this byte first, rejecting most non-matching slots without touching the key at all; only on a fingerprint match is the full key compared. Deletion uses backward-shift removal (shifting subsequent displaced elements back toward their ideal slots) rather than tombstones, which keeps the table clean without a separate cleanup pass.

The library ships two distinct memory layouts, and this is the most important thing to understand about it[^1]:

- **`unordered_flat_map`** — key/value pairs live inline in the bucket array. No indirection means the fastest access, but references and pointers to elements are **invalidated on rehash**, resizing causes allocation spikes, and large value types inflate the whole table.
- **`unordered_node_map`** — elements live in separately allocated nodes fed by a custom bulk allocator, giving **stable references and pointers** (but not stable iterators, matching `std::unordered_map`'s guarantee) and `const Key` in the pair. It is somewhat slower due to the extra indirection.
- **`unordered_map`** (and `unordered_set`) — a chooser alias that picks flat or node storage based on properties of the stored type. If you need a specific guarantee, name the concrete variant instead of relying on the alias.

`robin_hood::hash` provides tuned specializations for integer types and `std::string` and falls back to `std::hash` for everything else.

## Production Notes

- **Pointer/reference stability is layout-dependent.** The default `unordered_map` alias may resolve to the flat layout, where any insertion that triggers a rehash invalidates all references, pointers, and iterators. Code migrated from `std::unordered_map` that relied on stable element addresses (storing `&map[k]`, holding pointers across inserts) will break subtly. Use `unordered_node_map` explicitly when you need stability.

- **Bad hashes can throw, not just slow down.** Because the per-slot offset field is bounded, a pathologically poor hash function that produces very long probe sequences can exhaust the offset and cause the map to fail with `std::overflow_error` rather than merely degrading[^1]. In practice this is not observed with the built-in `robin_hood::hash`, but it is a real risk with a custom hash of poor quality or under adversarial (hash-flooding) input. This is not a DoS-hardened container.

- **Resize allocation spikes.** The flat layout allocates a new, larger contiguous array on growth and moves elements into it, so peak memory during a rehash is roughly the old plus the new table. For very large maps or large value types this transient can matter; the node layout's bulk allocator smooths allocation but does not eliminate the grow copy.

- **Iteration order is unspecified and unstable**, as with any open-addressing map. Do not depend on it, and expect it to change across insertions/rehashes.

- **Archived: no upstream fixes.** Since the project only accepts critical-bug PRs, any behavior you dislike is effectively permanent. New standards support, compiler-warning cleanups, and API additions will not land here — they land in unordered_dense.

## When to Use / When Not

**Use when:**
- You want a faster, lower-memory `std::unordered_map` replacement with an almost identical API and single-header integration.
- You are maintaining an existing codebase that already vendors robin_hood and it works.
- You can pick the layout deliberately: flat for raw speed, node for stable references.

**Avoid when:**
- You are starting new code — prefer martinus/unordered_dense, the maintained successor.
- You need pointer/reference stability and would otherwise reach for the default alias without checking which layout it resolves to.
- Your input is attacker-controlled and you need hash-flooding resistance; this map can throw on adversarial hashing.
- You require guaranteed iteration order or `std::unordered_map`'s exact node semantics throughout.

## Alternatives

- martinus/unordered_dense — the author's own successor (`ankerl::unordered_dense`); use this for any new project.
- abseil/abseil-cpp — `absl::flat_hash_map` (SwissTable) when you want Google's SIMD-probed, heavily production-tested map.
- Tessil/robin-map — another header-only Robin Hood open-addressing map when you want an actively maintained alternative with a similar design.
- greg7mdp/parallel-hashmap — `phmap` when you need sharded/concurrent access or lower peak memory during resize.
- ktprime/emhash — when you are optimizing purely for benchmark throughput and can tolerate a less conventional API.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2013-07 | Initial robin-hood-hashing repository[^3]. |
| 3.x line | 2019–2020 | Single-header `robin_hood.h` with flat/node layouts, info-byte probing. |
| 3.11.5 | 2021 | Final tagged release; version referenced by the Conan package[^1]. |
| archived | ~2023 | Development ends; author points users to unordered_dense[^2]. Last push 2023-05[^3]. |

## References

[^1]: Project README, `robin_hood::unordered_map & set`. https://github.com/martinus/robin-hood-hashing#readme
[^2]: README maintenance note directing users to the successor `ankerl::unordered_dense`. https://github.com/martinus/unordered_dense
[^3]: GitHub repository metadata (created 2013-07-27, last push 2023-05-01, archived), retrieved 2026-07. https://github.com/martinus/robin-hood-hashing

## Tags

cpp, hash-map, hash-table, robin-hood-hashing, header-only, single-file, data-structures, containers, stl-replacement, open-addressing, archived, no-dependencies
