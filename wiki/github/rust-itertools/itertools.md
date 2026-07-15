# rust-itertools/itertools

> Extra iterator adaptors, methods, free functions, and macros that extend Rust's standard `Iterator` — the de facto companion crate to `std::iter`.

[GitHub repo](https://github.com/rust-itertools/itertools) ·
[Documentation](https://docs.rs/itertools/) ·
[License: MIT OR Apache-2.0](https://github.com/rust-itertools/itertools/blob/master/README.md)

## Overview

`itertools` is a Rust library that fills the gaps in the standard library's iterator API. Rust's `std::iter::Iterator` is deliberately minimal — the standard library adds adaptors slowly and conservatively — so `itertools` collects the long tail of "things you constantly want an iterator to do" into one extension trait plus a handful of free functions and macros. It has been a near-universal dependency in the Rust ecosystem since before Rust 1.0; the crate was first published in 2015 and originates from work by Ulrik Sverdrup ("bluss")[^1], now maintained by a team of volunteers.

The core mechanism is the `Itertools` extension trait. Because it has a blanket implementation for every `Iterator`, a single `use itertools::Itertools;` adds ~100 methods (`chunk_by`, `unique`, `sorted`, `combinations`, `cartesian_product`, `join`, `dedup`, `tuple_windows`, `zip_eq`, and many more) to every iterator in scope, with no wrapper types visible to the caller. The crate stays intentionally at version `0.x` and has never committed to a `1.0`, which lets it keep evolving its surface as the standard library absorbs some of its adaptors.

The defining tension is that tension itself: `itertools` and `std` overlap and drift. Methods pioneered here (`Itertools::intersperse`, `array_chunks`-style windowing, `Itertools::flatten_ok`) periodically get standard-library equivalents, sometimes under the same name, occasionally causing ambiguity warnings when both traits are in scope[^2]. Using `itertools` is a bet that the convenience outweighs an eventual naming collision or a redundant dependency.

## Getting Started

Add it with Cargo:

```bash
cargo add itertools
```

```toml
[dependencies]
itertools = "0.15"
```

Bring the trait into scope and the adaptors light up on any iterator:

```rust
use itertools::Itertools;

fn main() {
    // group consecutive equal keys, then join and print
    let data = [1, 1, 2, 3, 3, 3, 4];
    let grouped = data
        .iter()
        .chunk_by(|&&x| x)                 // lazy run-length grouping
        .into_iter()
        .map(|(key, group)| format!("{key}x{}", group.count()))
        .join(", ");
    println!("{grouped}");                 // "1x2, 2x1, 3x3, 4x1"

    // cartesian product without nested loops
    for (a, b) in (0..2).cartesian_product("ab".chars()) {
        println!("{a}{b}");                // 0a 0b 1a 1b
    }

    // izip! zips three iterators into flat tuples
    for (x, y, z) in itertools::izip!(&[1, 2], &[3, 4], &[5, 6]) {
        println!("{x} {y} {z}");
    }
}
```

## Architecture / How It Works

`itertools` is structured around three delivery mechanisms:

1. **The `Itertools` trait** — an extension trait with a blanket `impl<T: Iterator + ?Sized> Itertools for T {}`. Methods return either a lazy adaptor struct (e.g. `Unique`, `Combinations`, `TupleWindows`) that implements `Iterator`, or a concrete value for terminal operations (`join`, `find_position`, `counts`, `collect_vec`). The adaptor structs live in the crate's public API so return types can be named, but callers usually just chain.
2. **Free functions** — `itertools::merge`, `kmerge`, `zip_eq`, `partition`, `process_results`, and others, for cases where a starting iterator does not read well as a method receiver.
3. **Macros** — `izip!` (n-ary zip producing flat tuples rather than nested pairs), `iproduct!` (n-ary cartesian product), and `chain!`.

Adaptors fall into two performance classes, and the split matters. **Streaming adaptors** (`interleave`, `intersperse`, `tuple_windows`, `dedup`, `batching`, `peeking_take_while`, `with_position`) are lazy and allocation-free: they pull one element at a time and compile down to roughly what a hand-written loop would. **Buffering adaptors** (`sorted`, `sorted_by`, `unique`, `combinations`, `permutations`, `powerset`, `group_map`, `multi_cartesian_product`, `k_smallest`) must collect or hash internally — `sorted` builds a `Vec` and sorts it, `unique` maintains a `HashSet`, `combinations` holds a buffer of indices. These are convenient one-liners for what would otherwise be several lines, but they are not free, and calling them in a hot loop over large inputs is the most common way `itertools` shows up in a profile.

The `Either` type used by adaptors like `partition_map` lives in the separate `either` crate (also under the `rust-itertools` org) and is re-exported. `no_std` is supported: disabling default features drops `use_std`, leaving a smaller surface; enabling `use_alloc` restores the adaptors that need heap allocation without requiring the full standard library.

## Production Notes

**Buffering adaptors allocate — know which ones.** `sorted()` is the classic trap: it is a terminal-ish adaptor that collects the entire iterator into a `Vec` and sorts it, so `.sorted().take(3)` sorts everything to return three elements. Use `k_smallest(3)` when you only need a prefix. Similarly `unique()` holds every seen value in a hash set for the life of the iterator; over an unbounded or very large stream this is a memory leak in slow motion.

**The `intersperse` / std collision.** The standard library gained an unstable `Iterator::intersperse` on nightly. When both that feature and `use itertools::Itertools` are in scope, calls to `.intersperse(...)` become ambiguous and emit an `unstable_name_collisions` warning; the intended fix is fully-qualified syntax (`Itertools::intersperse(iter, sep)`) until the situation stabilizes[^2]. This is the crate's most visible real-world friction point and a recurring source of confused issue reports.

**`group_by` was renamed to `chunk_by`.** Because the semantics only group *consecutive* equal keys (like Unix `uniq`, not SQL `GROUP BY`), and to avoid confusion with the standard library's slice `chunk_by`, `itertools` deprecated `group_by` in favor of `chunk_by`[^3]. Code upgrading across that boundary will hit deprecation warnings. The grouping iterator also borrows from the outer `ChunkBy` value, so you must bind it to a variable and call `.into_iter()` — a lifetime dance that surprises newcomers.

**MSRV and version churn.** `itertools` tracks a conservative Minimum Supported Rust Version and treats bumping it as a semver-relevant change, but because the crate is `0.x`, every minor release (`0.12` → `0.13` → `0.14` → `0.15`) is a breaking release under Cargo's `0.x` rules. Ecosystem crates that expose `itertools` types in their public API force downstream users onto a matching major-minor, and it is common for a large dependency tree to compile two or three `itertools` versions simultaneously. Pin loosely (`"0.15"`) and expect periodic coordinated upgrades.

**`zip_eq` panics.** Unlike std's `zip`, which stops at the shorter iterator, `zip_eq` panics if the lengths differ. That is the point — it is a correctness assertion — but it means `zip_eq` is unsuitable for streams whose lengths you do not control.

## When to Use / When Not

**Use when:**
- You reach for an iterator operation std does not have (`chunk_by`, `unique`, `cartesian_product`, `tuple_windows`, `join`, `dedup`, `with_position`).
- You want readable one-liners for combinatorics (`combinations`, `permutations`, `powerset`) on modest input sizes.
- You are formatting iterators into strings (`join`, `format`, `format_with`) and want to avoid intermediate `Vec`s.

**Avoid when:**
- The equivalent now exists in stable `std` (e.g. `Iterator::intersperse` where stabilized, slice `chunk_by`) — one fewer dependency, no collision risk.
- You need parallelism — `itertools` is single-threaded; reach for Rayon.
- You are in a hot path and would call a buffering adaptor (`sorted`, `unique`, `permutations`) per iteration — a hand-written loop or a `k_smallest`-style bounded variant is cheaper.

## Alternatives

- rayon-rs/rayon — use when the workload is CPU-bound and parallelizable; provides parallel iterator adaptors instead of sequential ones.
- rust-itertools/either — use directly when you only need the `Either` enum that `itertools` re-exports, without the rest of the crate.
- rust-itertools/itertools-num — numeric-focused helpers (e.g. `linspace`, cumulative sums); lightly maintained, useful only for those specific needs.
- Rust `std::iter` — prefer the standard library when the method you want already exists there (`chunk` windows, `flatten`, `scan`), to avoid the extra dependency and any name collision.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015 | First published to crates.io; predates Rust 1.0[^1]. |
| 0.8 | ~2019 | Broad adaptor set established; widely depended upon. |
| 0.10 | 2021 | `Either` moved to the separate `either` crate; continued API growth. |
| 0.12 | ~2023 | Adaptor additions; MSRV maintenance. |
| 0.13 | ~2024 | `group_by` deprecated in favor of `chunk_by`[^3]. |
| 0.14 | ~2024 | Continued adaptor/API refinement. |
| 0.15 | 2025 | Current line; `use itertools::Itertools;`[^4]. |

## References

[^1]: itertools on crates.io — original author Ulrik Sverdrup ("bluss"), maintained by the rust-itertools team. https://crates.io/crates/itertools
[^2]: `unstable_name_collisions` between `Itertools::intersperse` and the standard library's nightly `Iterator::intersperse`; discussed across the crate's issue tracker. https://github.com/rust-itertools/itertools/issues
[^3]: `chunk_by` (formerly `group_by`) documentation and rename rationale. https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.chunk_by
[^4]: itertools API documentation. https://docs.rs/itertools/

## Tags

rust, iterators, iterator-adaptors, functional, combinatorics, standard-library-extension, no-std, cargo-crate, utility-library, lazy-evaluation
