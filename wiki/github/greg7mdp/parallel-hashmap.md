# greg7mdp/parallel-hashmap

> Header-only C++ hash maps and btrees derived from Abseil's Swiss tables, with a "parallel" sharded variant for large tables and optional internal locking.

[GitHub repo](https://github.com/greg7mdp/parallel-hashmap) ·
[Official website](https://greg7mdp.github.io/parallel-hashmap/) ·
[License: Apache-2.0](https://github.com/greg7mdp/parallel-hashmap/blob/master/LICENSE)

## Overview

parallel-hashmap (namespace `phmap`, often "phmap") is a set of header-only associative containers for C++11 and later. It reimplements Google Abseil's open-addressed "Swiss table" hash maps — with modifications — plus Abseil's btree ordered containers, and adds a distinctive `parallel_*` family that shards a single logical map into 16 (or 2^N) internal submaps[^1]. It has grown from an offshoot of the author's earlier `sparsepp` into a widely vendored dependency; it currently sits around 3,200 stars and is actively maintained but in maintenance/bugfix mode (last push mid-2026), with the maintainer steering new development toward a C++20 successor, `gtl`[^2].

The core value proposition is memory efficiency and speed relative to `std::unordered_map`. Swiss tables store values inline in a single array and use SSE2 to probe 16 slots at once, staying fast up to 87.5% load factor. The `parallel` variant's real purpose is not just concurrency: by sharding, it cuts the *peak* memory spike during a resize (only one submap rehashes at a time) and, as a side effect, enables fine-grained internal locking. The defining tension is the same one Abseil users face — inline storage means pointers and iterators into `flat_*` and btree containers are invalidated by mutation, unlike `std::unordered_map`'s node stability. You trade reference stability for cache locality, and must reach for the `node_*` variants when that trade is unacceptable.

## Getting Started

Header-only. Copy the `parallel_hashmap/` directory into your project and add it to the include path — there is nothing to build.

```cpp
#include <parallel_hashmap/phmap.h>
using phmap::flat_hash_map;

int main() {
    flat_hash_map<std::string, int> ages = {{"tom", 35}, {"jane", 32}};
    ages["bill"] = 40;
    if (auto it = ages.find("jane"); it != ages.end())
        return it->second;      // heterogeneous + std API, drop-in for unordered_map
    return 0;
}
```

For thread-safe concurrent access, use a `parallel_*` table with a mutex as the last template parameter and the callback-based APIs, which hold the submap lock across the operation:

```cpp
#include <parallel_hashmap/phmap.h>
phmap::parallel_flat_hash_map<
    int, int, phmap::priv::hash_default_hash<int>,
    phmap::priv::hash_default_eq<int>,
    std::allocator<std::pair<const int, int>>, 4, std::mutex> m;

m.lazy_emplace_l(42,
    [](auto& v) { v.second++; },                 // key exists: mutate under lock
    [](const auto& ctor) { ctor(42, 1); });      // key absent: construct under lock
```

## Architecture / How It Works

The header `parallel_hashmap/phmap.h` provides eight hash containers along two independent axes:

- **flat vs node.** `flat_*` stores `value_type` plus one metadata byte directly in the bucket array (cache-friendly, less memory, but moves values on resize → no pointer stability). `node_*` stores a pointer to a heap node plus one metadata byte, preserving pointer/reference stability at the cost of an indirection and ~`sizeof(void*)` overhead per slot.
- **single vs parallel.** `parallel_*` maps hold 2^N submaps (default template arg `N=4` → 16 submaps). A high bit range of the hash selects the submap; the rest indexes within it. This bounds resize peak memory to roughly `1/(2·submaps)` of the single-map spike and localizes lock contention.

`btree.h` adds `btree_set/map/multiset/multimap`, direct ports of Abseil's btrees — multiple values per node, far more cache-friendly and memory-lean than the STL's red-black trees, but with weaker iterator guarantees.

Notable deviations from upstream Abseil[^3]:

- **Default hash is `std::hash`, not `absl::Hash`.** Define `PHMAP_USE_ABSL_HASH` to switch.
- **Hash mixing is on by default.** phmap internally mixes the user hash to tolerate poor-entropy hash functions; Abseil does not do this the same way. Disable with `PHMAP_DISABLE_MIX=1` (not recommended).
- **Iteration order is deterministic by default.** Abseil randomizes a per-table seed as a DoS hardening measure; phmap disables this unless you define `PHMAP_NON_DETERMINISTIC=1`. Convenient for debugging, but means untrusted-key-controlled tables lose that protection.
- No new string types — `std::string_view` and anything with a `std::hash` specialization work directly. `phmap_fwd_decl.h` enables forward declaration (except maps with pointer keys). `phmap_utils.h` provides `phmap::HashState().combine(...)` for composing hashes of user types.

## Production Notes

- **Iterator/pointer invalidation is the #1 footgun.** For `flat_*` and all btrees, any insert that triggers a rehash (or any btree mutation at all) can move stored values — cached pointers become dangling. `node_*` avoids this for hash maps. Code migrating from `std::unordered_map`/`std::map` that relies on node stability will break subtly, not at compile time.
- **The parallel mutex does not protect returned iterators/references.** `if_contains`, `modify_if`, `try_emplace_l`, `lazy_emplace_l` are safe because the callback runs under the submap lock. But a plain `find()`/`operator[]` returns an iterator that is *not* guarded — using it while another thread writes is a data race. Concurrent designs must stay inside the callback APIs.
- **Dump/load is fast but restrictive.** The binary `dump`/`load` path (flat tables only, `std::trivially_copyable` values) serializes the raw array ~10× faster than element-wise, at 10–60% extra disk size — and the dump is layout/architecture-specific, not a portable format.
- **`parallel_*` is the wrong default for many small maps.** Its wins (resize peak memory, threading) apply to a *few very large* tables. If you have many small tables, the 16-submap overhead is pure cost; use the non-parallel variants.
- **Compiler/ABI surface.** Header-only means every translation unit compiles the templates; heavy use inflates build times. Behavior depends on the macros above being defined *consistently* across all includes — mismatched `PHMAP_USE_ABSL_HASH`/`PHMAP_NON_DETERMINISTIC` across TUs is an ODR hazard.
- **Successor drift.** New features land in `gtl` (C++20), not here. phmap remains supported for bugfixes, but the maintainer explicitly recommends migrating if you can target C++20[^2]. Plan for it if you want long-term feature parity.

## When to Use / When Not

**Use when:**
- You want a faster, leaner drop-in for `std::unordered_map`/`set` and can tolerate flat-table iterator invalidation (or use `node_*`).
- You hold one or a few maps with very large element counts and want bounded resize memory spikes or built-in sharded locking.
- You need a header-only dependency with no build step and broad compiler support (C++11 up).

**Avoid when:**
- You require `std::unordered_map`-style pointer/reference stability everywhere and can't switch to `node_*`.
- You can target C++20 and want the actively developed line — use `gtl` instead.
- You need cryptographic/DoS resistance for attacker-controlled keys out of the box (re-enable randomization and pick a strong hash deliberately).
- You need a portable on-disk serialization format — the dump/load feature is not one.

## Alternatives

- greg7mdp/gtl — same author, same tables, C++20; where new development happens. Prefer it when you can require C++20.
- abseil/abseil-cpp — the upstream Swiss-table source; use when you already depend on Abseil or want `absl::Hash` and its DoS hardening.
- martinus/unordered_dense — header-only, often competitive or faster for many workloads with a simpler design; use when you want a single lightweight header.
- Tessil/robin-map — robin-hood hashing, strong all-round performer; use when you want predictable probe behavior without SSE dependence.
- greg7mdp/sparsepp — the author's earlier map optimized for minimal memory over speed; use when memory footprint dominates and speed is secondary.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-03 | Repo created; Swiss-table maps ported from Abseil with phmap modifications[^1]. |
| btree add | 2020 | Abseil btree containers ported in (`btree_set/map/...`). |
| — | 2020-11 | Chinese README translation contributed; broad ecosystem adoption underway. |
| gtl split | ~2022 | C++20 successor `gtl` created; new development redirected there[^2]. |
| maintenance | 2026-06 | Still receiving fixes (last push 2026-06); users steered toward gtl for C++20. |

## References

[^1]: Repository README — design overview and Abseil derivation. https://github.com/greg7mdp/parallel-hashmap/blob/master/README.md
[^2]: greg7mdp/gtl — C++20 successor project the maintainer recommends. https://github.com/greg7mdp/gtl
[^3]: README, "Changes to Abseil's hashmaps" — default hash, hash mixing, determinism, string types. https://github.com/greg7mdp/parallel-hashmap#changes-to-abseils-hashmaps

## Tags

cpp, c-plus-plus, hash-map, header-only, data-structures, swiss-table, abseil, btree, concurrency, memory-efficiency, unordered-map
