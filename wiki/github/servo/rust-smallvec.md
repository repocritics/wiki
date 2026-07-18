# servo/rust-smallvec

> "Small vector" optimization for Rust: a `Vec<T>` drop-in that stores up to N
> elements inline and only heap-allocates past that.

[GitHub repo](https://github.com/servo/rust-smallvec) ·
[Documentation](https://docs.rs/smallvec/) ·
[License: Apache-2.0](https://github.com/servo/rust-smallvec/blob/v2/LICENSE-APACHE)
(published on crates.io as MIT OR Apache-2.0 dual license[^1])

## Overview

`smallvec` is the canonical small-vector-optimization (SVO) crate for Rust,
extracted from Mozilla's Servo browser engine project in 2015. It provides
`SmallVec`, a growable vector that keeps its first N elements inside the struct
itself — typically on the stack or inline in a parent struct — and transparently
moves ("spills") to a heap allocation when it grows past N. For workloads with
many short vectors that usually stay small (AST node children, style rule
matches, argument lists), this eliminates the allocator round-trip that
dominates `Vec`'s cost at small sizes.

Its 1,673 GitHub stars undersell its footprint: `smallvec` is a transitive
dependency of a large fraction of the Rust ecosystem and is used heavily inside
`rustc` itself and Servo. It is one of the most-downloaded crates on
crates.io[^1], which also means its heavy use of `unsafe` has real blast radius —
it has shipped three RUSTSEC memory-safety advisories over its history (see
Production Notes).

The defining tradeoff: inline storage is not free. A `SmallVec<[T; N]>` is at
least as large as its inline buffer, moves copy that buffer, and every element
access branches on whether the vector has spilled. SVO is a profile-guided
optimization, not a default; used blindly, `SmallVec` is routinely slower than
`Vec`. The project is currently mid-transition: the default branch is `v2`, an
unreleased const-generics rewrite, while the published stable line (1.x) lives
on the `v1` branch[^2].

## Getting Started

```bash
cargo add smallvec        # stable 1.x line
```

```rust
use smallvec::{smallvec, SmallVec};

// Up to 4 elements inline; no heap allocation until the fifth.
let mut v: SmallVec<[i32; 4]> = smallvec![1, 2, 3];
v.push(4);
assert!(!v.spilled());

v.push(5);                // fifth element spills to the heap
assert!(v.spilled());

// Deref<Target = [i32]>: all slice methods and indexing work.
v.sort();
assert_eq!(v[0], 1);
```

The 1.x API takes an array type parameter (`SmallVec<[T; N]>`). The unreleased
v2 API switches to const generics (`SmallVec<T, N>`), which is what the
repository README currently shows — do not copy README examples against the
1.x crate[^2].

## Architecture / How It Works

A 1.x `SmallVec` is a `capacity: usize` field plus a data payload that is either
an inline buffer of `[T; N]` or a `(pointer, length)` pair for a heap
allocation. The `capacity` field does double duty: while inline it stores the
length; once spilled it stores the heap capacity, and `spilled()` is derived by
comparing it against the inline capacity. Growth past N allocates on the heap,
moves the inline elements over, and from then on the type behaves like a
`Vec` — including amortized reallocation on further growth. `shrink_to_fit` can
move data back inline if the length fits.

Notable design points:

- **`Deref<Target = [T]>`** — `SmallVec` is a slice for all read paths, so the
  API surface tracks `Vec` closely and most call sites are drop-in.
- **The `union` feature** — replaces the internal tagged enum with an untagged
  union, shrinking the struct by one word. Historically nightly-only, now
  usable on stable rustc; still off by default in 1.x for MSRV reasons.
- **Feature flags** — `serde` (serialization), `write` (`std::io::Write` for
  `SmallVec<[u8; N]>`), and const-generics compatibility shims on the 1.x line.
- **v2 rewrite** — drops the `Array` helper trait for real const generics,
  reworks the internal layout, and trims accumulated API cruft; tracked in
  issues #183, #240, and #284[^2]. Development activity continues on the `v2`
  default branch (last push 2026-06), with 1.x bug fixes on `v1`.

The implementation is dense `unsafe` code by necessity — raw-pointer element
management over a maybe-inline, maybe-heap buffer — which is precisely where its
historical CVEs came from.

## Production Notes

**Measure before adopting.** The classic failure mode is sprinkling `SmallVec`
everywhere and losing throughput: large N inflates every struct that embeds the
vector, moves become O(N) memcpys instead of pointer swaps, and the
spilled-check branch sits on every hot access. SVO pays off when profiling
shows allocator pressure from many short-lived, usually-small vectors — the
Servo/rustc workload it was built for. If the vector almost always exceeds N,
you pay both costs.

**Choosing N.** Pick N from observed size distributions, not intuition, and
keep `size_of` in mind: a `SmallVec<[u64; 32]>` in a frequently-moved struct is
a 256-byte memcpy waiting to happen. Small N values (2–8) for element types a
word or two wide are the common sweet spot.

**Security history.** Three RUSTSEC advisories: double-free/use-after-free in
`SmallVec::grow` (RUSTSEC-2019-0009, 0.6.x)[^3], memory corruption in `grow`
(RUSTSEC-2019-0012)[^4], and a buffer overflow in `insert_many` fixed in 1.6.1
and 0.6.14 (RUSTSEC-2021-0003)[^5]. All are long-patched, but pin ≥ 1.6.1 and
run `cargo audit` — old lockfiles in long-lived projects have resurfaced these.

**The v1/v2 split.** Everything stable on crates.io is 1.x; 2.0 exists only as
alpha pre-releases[^1]. The GitHub default branch, README, and open PRs target
the unreleased v2 API, which is a recurring source of confusion when reading
the repo against the crate you actually depend on. Migration will be a breaking
type-signature change (`SmallVec<[T; N]>` → `SmallVec<T, N>`) across every
public API that exposes the type.

**API drift from `Vec`.** Coverage of `Vec`'s API is close but not complete,
and some methods (`insert_many`, `from_buf`, `into_inner`) have no `Vec`
equivalent. Code that is generic over "some vector" usually ends up on
`Deref<Target = [T]>` plus `Extend`/`push` rather than a shared trait.

## When to Use / When Not

**Use when:**
- Profiling shows allocation churn from many short vectors that are usually
  small (parsers, compilers, ECS component lists, per-request scratch space).
- You control the size distribution and can pick N from data.
- You want `Vec` semantics — growable, heap fallback — not a hard capacity cap.

**Avoid when:**
- You have not profiled; `Vec` is the correct default in Rust.
- Vectors routinely exceed N — you pay the branch and the bigger struct for
  nothing.
- The capacity is genuinely fixed — a fixed-capacity type is simpler and
  smaller.
- You need to minimize `unsafe` in your dependency tree — safer alternatives
  exist at a small cost (see below).

## Alternatives

- bluss/arrayvec — fixed-capacity vector that never heap-allocates; use it when
  overflow should be an error, not a spill.
- Lokathor/tinyvec — 100%-safe-code SVO vector; use it when auditability beats
  raw performance (requires `T: Default`).
- Gankra/thin-vec — vector that is one pointer wide with length/capacity in the
  heap header (Gecko's ThinVec); use it when struct size matters more than
  avoiding allocation.
- rust-embedded/heapless — statically allocated data structures for `no_std`;
  use it when there is no allocator at all.
- Standard `Vec` — use it unless a profiler tells you otherwise.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015 | Extracted from Servo; repo created 2015-04. |
| 1.0 | 2019-11 | Stable 1.x line; `union` and const-generics work landed as opt-in features across later 1.x releases. |
| 1.6.1 | 2021-01 | Security release for `insert_many` buffer overflow (RUSTSEC-2021-0003)[^5]. |
| 2.0 (unreleased) | — | Const-generics rewrite on the `v2` default branch; alpha pre-releases on crates.io; active as of 2026-06[^2]. |

## References

[^1]: smallvec on crates.io — license and release listing. https://crates.io/crates/smallvec
[^2]: servo/rust-smallvec README (v2 branch note) and tracking issues. https://github.com/servo/rust-smallvec/issues/183
[^3]: RUSTSEC-2019-0009 — double-free and use-after-free in `SmallVec::grow()`. https://rustsec.org/advisories/RUSTSEC-2019-0009.html
[^4]: RUSTSEC-2019-0012 — memory corruption in `SmallVec::grow()`. https://rustsec.org/advisories/RUSTSEC-2019-0012.html
[^5]: RUSTSEC-2021-0003 — buffer overflow in `SmallVec::insert_many`. https://rustsec.org/advisories/RUSTSEC-2021-0003.html

## Tags

rust, data-structures, small-vector-optimization, performance, memory-allocation, no-std, servo, library, unsafe-heavy, systems-programming
