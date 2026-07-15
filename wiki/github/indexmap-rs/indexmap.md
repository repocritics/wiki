# indexmap-rs/indexmap

> A hash table that keeps insertion order and lets you address entries by position, built on the same SwissTable core as Rust's std `HashMap`.

[GitHub repo](https://github.com/indexmap-rs/indexmap) ·
[Docs](https://docs.rs/indexmap/) ·
[License: Apache-2.0](https://github.com/indexmap-rs/indexmap/blob/main/LICENSE-APACHE)

## Overview

`indexmap` provides two containers, `IndexMap` and `IndexSet`, whose iteration
order is independent of the hash function and instead follows insertion order.
It was inspired by Python 3.6's compact `dict`, which brought ordered iteration
and lower memory overhead to that language[^1]. The crate started life under the
name `ordermap` and was renamed to `indexmap` to reflect that entries can also be
looked up by a numeric position, not just by key[^2].

The audience is anyone who reaches for a `HashMap` but needs deterministic
iteration — serializers that must preserve field order, config parsers, compiler
symbol tables, anything that dumps a map to a human. It is one of the most
widely-depended-on crates in the Rust ecosystem: `serde_json`'s `preserve_order`
feature stores objects in an `IndexMap`, and the `toml` crate uses it to keep
table order stable across a round-trip.

The defining tension is the two-level layout. `IndexMap` is a hash table of
*indices* pointing into a dense `Vec` of key-value pairs. That makes iteration
and index access fast and ordering cheap, but every keyed lookup touches two
memory regions (the index table, then the entry vector), so single random
lookups are somewhat slower and less cache-friendly than a plain `HashMap`. You
trade a little lookup latency and a `remove` footgun (below) for order and
positional access.

## Getting Started

```bash
cargo add indexmap
# optional features: cargo add indexmap --features serde,rayon
```

```rust
use indexmap::IndexMap;

let mut map = IndexMap::new();
map.insert("c", 3);
map.insert("a", 1);
map.insert("b", 2);

// Iteration follows insertion order, not hash order.
for (k, v) in &map {
    println!("{k} = {v}");        // c=3, a=1, b=2
}

// Address entries by position as well as by key.
assert_eq!(map.get("a"), Some(&1));
assert_eq!(map.get_index(0), Some((&"c", &3)));
assert_eq!(map.get_index_of("b"), Some(2));

// swap_remove is O(1) but moves the LAST entry into the gap.
map.swap_remove("c");
assert_eq!(map.get_index(0), Some((&"b", &2)));   // order changed
```

## Architecture / How It Works

The internal shape, roughly, is *a hash table of entry indices plus a vector of
entries*[^3]. The hash lookup uses `hashbrown` — the same SwissTable
implementation that backs the standard library's `HashMap`[^4] — but instead of
storing keys and values in the table's buckets, buckets store a `usize` index
into a separate `Vec<Bucket<K, V>>` that holds the actual pairs in insertion
order.

Consequences fall out directly from that layout:

- **Iteration is fast.** It is a linear scan over the dense entry vector, with no
  probing and no empty slots to skip.
- **Positional access is O(1).** `get_index`, `get_index_of`, and slicing
  (`map::Slice`) work because the vector is the source of truth for order.
- **Lookups pay for indirection.** The 7-bit SIMD probe finds the index quickly,
  but reading the key to confirm the match, and then the value, means a second
  jump into the entry vector. Under cache pressure this is the crate's weak spot.
- **Removal has two flavors.** `swap_remove` is O(1): it swaps the last entry
  into the vacated slot, which disrupts order. `shift_remove` preserves relative
  order but is O(n) because it shifts every following entry down and rewrites
  their indices. The bare `remove` method is deprecated precisely because its
  swap-semantics surprised people.

The `Equivalent` trait (re-exported from the `equivalent` crate) lets lookups use
borrowed or alternate key types, and `Entry`-style APIs mirror the standard
library. Default hashing uses `RandomState` when the `std` feature is on;
`no_std` builds require `alloc` and a supplied hasher.

## Production Notes

- **`remove` is a footgun.** If order matters after deletion, you must choose
  `shift_remove` explicitly; `swap_remove` (and the deprecated `remove`) will
  reorder silently. This is the single most common bug filed against downstream
  crates that adopt `IndexMap`.
- **Not a drop-in for `HashMap` on hot lookups.** If your workload is
  lookup-dominated with no ordering requirement, the extra indirection is pure
  cost — benchmark before swapping. `indexmap` was tested as rustc's internal map
  in PR45282 and came out roughly on par overall, not a clear win[^5].
- **`serde` field order.** Enabling `preserve_order` on `serde_json` transparently
  swaps its object type to `IndexMap`; this is the standard way to keep JSON key
  order across parse-then-serialize, but it changes the public type of
  `Value::Object`, which can ripple through code that pattern-matches on it.
- **MSRV moves with releases.** The crate tracks a relatively recent minimum Rust
  version (1.85+ as of the 2.14 line); pinning an older toolchain means pinning an
  older `indexmap`. The 2.0 release (2023) raised the floor to 1.64 and reworked
  feature flags[^6].
- **Feature-gate the extras.** `serde`, `rayon` (parallel iteration/sort),
  `borsh`, `arbitrary`, and `quickcheck` are all optional and off by default;
  pulling `rayon` in particular adds a substantial dependency subtree.
- **Capacity is approximate.** `reserve_exact` and `try_reserve` apply to the
  entry vector, but the underlying raw table still follows its own load-factor
  rules, so exact capacity control is not fully exact.

## When to Use / When Not

**Use when:**
- You need deterministic iteration order tied to insertion.
- You want to look up entries by numeric position as well as by key.
- You are serializing maps and want stable, human-predictable output.
- You want ordered behavior without the pointer-chasing of a linked hash map.

**Avoid when:**
- The workload is dominated by random single-key lookups and order is irrelevant —
  a plain `HashMap` or raw `hashbrown` will be more cache-friendly.
- You need the strongest ordering guarantees on removal by default (reach for
  `ordermap`, which wraps this crate with shift-remove semantics up front).
- The map is static and known at compile time — a perfect-hash map is faster.

## Alternatives

- rust-lang/hashbrown — the SwissTable that `indexmap` and std both build on; use it directly when you want the fast table and no ordering.
- indexmap-rs/ordermap — thin wrapper over `indexmap` with stronger ordering guarantees (order-preserving removal by default); use it when order-after-remove must be automatic.
- rust-phf/rust-phf — compile-time perfect hash maps; use for static, read-only key sets where lookup speed is paramount.
- contain-rs/linked-hash-map — older linked-list-backed ordered map; use only for legacy code, as iteration is slower and less compact.
- The standard library `std::collections::HashMap` — use when you need neither ordering nor positional access and want zero extra dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.4.1 (as `indexmap`) | 2018-02-14 | Renamed from `ordermap` to `indexmap`[^2]. |
| 1.0.0 | 2018-03-11 | First 1.x release. |
| 1.9.x | 2022 | Mature 1.x line; `hashbrown`-backed table. |
| 2.0.0 | 2023-06-23 | MSRV 1.64, feature-flag rework, `map::Slice`, `serde-1` removed, `get_index_mut` signature change[^6]. |
| 2.7.0 | 2024-11-30 | Continued 2.x additions. |
| 2.14.0 | 2026-04-09 | Current line; MSRV 1.85+. |

## References

[^1]: Python 3.6 release notes — compact, ordered `dict` implementation. https://docs.python.org/3/whatsnew/3.6.html#whatsnew36-compactdict
[^2]: indexmap README — "originally released under the name `ordermap`, but it was renamed to `indexmap`." https://github.com/indexmap-rs/indexmap/blob/main/README.md
[^3]: indexmap docs.rs — data-structure overview. https://docs.rs/indexmap/latest/indexmap/
[^4]: hashbrown — Rust port of Google's SwissTable, used by std `HashMap`. https://github.com/rust-lang/hashbrown
[^5]: rust-lang/rust PR #45282 — trial of an indexmap-style table inside rustc. https://github.com/rust-lang/rust/pull/45282
[^6]: indexmap RELEASES.md — 2.0.0 (2023-06-23) changelog. https://github.com/indexmap-rs/indexmap/blob/main/RELEASES.md

## Tags

rust, hash-table, hashmap, ordered-map, insertion-order, data-structures, hashbrown, swisstable, serde, no-std
