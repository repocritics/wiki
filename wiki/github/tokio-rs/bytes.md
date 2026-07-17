# tokio-rs/bytes

> A reference-counted, cheaply-cloneable contiguous byte buffer for zero-copy network and I/O code.

[GitHub repo](https://github.com/tokio-rs/bytes) ·
[Documentation](https://docs.rs/bytes) ·
[License: MIT](https://github.com/tokio-rs/bytes/blob/master/LICENSE)

## Overview

`bytes` is a small Rust crate maintained under the Tokio project that provides two
buffer types — `Bytes` and `BytesMut` — plus two traits, `Buf` and `BufMut`, for
reading from and writing into byte sequences with a cursor abstraction. It is one of
the foundational crates of the async Rust I/O stack: `hyper`, `tonic`, `h2`, `prost`,
`tokio-util`, and most of the networking ecosystem take and return `Bytes` at their
API boundaries. Despite ~2.2k stars[^1] it is a de facto standard — its true reach is
measured by reverse dependencies (tens of thousands of crates), not stars.

The single idea that defines the crate is **cheap, safe sharing of an owned byte
region**. `Bytes` is an immutable, atomically reference-counted view into a heap (or
static) allocation. Cloning a `Bytes` is an `O(1)` refcount bump, not a memory copy,
and slicing (`slice`, `split_to`, `split_off`) produces non-overlapping views that
share the same backing allocation without copying. This is what makes zero-copy
protocol parsing practical: a parser can hand out many independently-owned sub-slices
of one received packet, each cleaned up automatically when the last owner drops.

The defining tradeoff is **retention**. Because every slice keeps the whole backing
allocation alive, holding a 20-byte `Bytes` sliced out of a 64 KiB read buffer keeps
all 64 KiB resident until that slice drops. The API is fast and safe but quietly
trades memory for the absence of copies — a tradeoff you must opt out of deliberately.

## Getting Started

```toml
[dependencies]
bytes = "1"
```

```rust
use bytes::{Bytes, BytesMut, Buf, BufMut};

// Build into a growable, uniquely-owned buffer.
let mut buf = BytesMut::with_capacity(64);
buf.put_u8(b'h');
buf.put(&b"ello world"[..]);

// Freeze into an immutable, shareable Bytes (no copy — same allocation).
let data: Bytes = buf.freeze();

// O(1) split: `head` and `data` share the allocation, disjoint ranges.
let mut data = data;
let head = data.split_to(5);          // "hello"
assert_eq!(&head[..], b"hello");
assert_eq!(&data[..], b" world");

// Cursor reads via the Buf trait.
let mut cur = &b"\x00\x01\x02\x03"[..];
let n = cur.get_u32();                // advances the cursor by 4
assert_eq!(n, 0x0001_0203);
```

For `no_std`, disable default features (`default-features = false` drops the `std`
feature). Serde support is behind the `serde` feature; both track the MSRV of their
respective dependencies rather than the crate's own.[^2]

## Architecture / How It Works

`Bytes` is not a single representation — it is a fat pointer plus a `&'static Vtable`.
Internally it stores a data pointer, a length, and a vtable whose `clone`, `drop`, and
`to_mut` function pointers implement the sharing strategy for whichever kind of storage
the value currently holds. This lets one type transparently represent several cases:

- **Static** — `Bytes::from_static(b"...")` wraps a `&'static [u8]` with a no-op
  clone/drop vtable. No allocation, no refcount.
- **Vec-backed** — `Bytes::from(vec)` takes ownership of a `Vec<u8>` with no copy. The
  first `clone` promotes it to a shared atomic representation (a one-time cost) so both
  owners can drop independently.
- **Shared** — an `Arc`-like atomic refcount guarding the allocation; `clone` is an
  atomic increment, `drop` a decrement that frees only at zero.

`BytesMut` is the mutable, single-owner counterpart. It encodes its kind (`Vec`-like
vs. shared) in low bits of a pointer and supports two things `Bytes` cannot: in-place
mutation and capacity reclamation. When you `split_off`/`split_to` a `BytesMut` and
later drop the other half, a subsequent `reserve` can reuse the vacated space in the
same allocation instead of reallocating. Adjacent halves can be recombined with
`unsplit` in `O(1)`. `freeze()` converts a `BytesMut` into a `Bytes` without copying.

`Buf` and `BufMut` are cursor traits, not containers. `Buf` exposes `remaining()`,
`chunk()`, and `advance()`; the `get_*` helpers (`get_u32`, `get_uint`, …) are default
methods on top of those. `&[u8]`, `Bytes`, `BytesMut`, `VecDeque<u8>`, and chained
buffers (`Buf::chain`) all implement them, which is why generic I/O code in `hyper`/
`tonic` is written against `impl Buf` rather than a concrete type. The traits also
model potentially non-contiguous buffers (vectored I/O) via `chunks_vectored`.

## Production Notes

- **Retention footgun (the big one).** Slicing a small `Bytes` out of a large read
  keeps the entire backing allocation alive. In long-lived caches or connection state,
  this silently pins megabytes. If you intend to retain a small slice long-term, copy
  it out: `Bytes::copy_from_slice(&big[range])` allocates a right-sized buffer and lets
  the large one drop.
- **`clone` is cheap, `copy_from_slice` is not.** These look similar but have opposite
  cost/retention profiles. `clone` = refcount bump, shares memory. `copy_from_slice` =
  allocation + memcpy, independent memory. Choose per the retention concern above.
- **`BytesMut` capacity behavior is subtle.** Growth via `reserve`/`put_*` may or may
  not reallocate depending on whether split-off siblings are still alive. Two `BytesMut`
  halves of the same allocation cannot both grow into the shared region; the second one
  to need space reallocates. Preallocate with `with_capacity` on hot paths.
- **Immutability is enforced, not advisory.** `Bytes` has no safe mutation API by
  design; to edit shared data you must go through `BytesMut` (via `try_into_mut` /
  `into()`), which only succeeds cheaply when you hold the sole reference — otherwise it
  copies. Code that clones a `Bytes` widely and then wants to mutate will pay for copies.
- **Atomic refcounts cost on extreme fan-out.** Sharing across threads uses atomic
  operations; workloads that clone/drop the same `Bytes` millions of times across cores
  can see contention. It is still far cheaper than copying, but it is not free.
- **MSRV and features.** The crate itself has a conservative MSRV, but enabling `serde`
  or `extra-platforms` (for targets without atomic CAS, via `portable-atomic`) ties your
  effective MSRV to those dependencies.[^2] Pin accordingly in constrained builds.
- **API stability.** Since 1.0 the crate follows semver strictly and has been unusually
  stable; upgrades within the 1.x line are routine and rarely require code changes.[^3]

## When to Use / When Not

**Use when:**
- You parse or forward network data and want zero-copy sub-slicing with automatic cleanup.
- You need to share a read-only byte region across tasks or threads without copying.
- You are writing to (or interoperating with) `hyper`, `tonic`, `h2`, `prost`, or
  `tokio-util`, which speak `Bytes`/`Buf` natively.
- You want a single buffer type that can be static, `Vec`-backed, or shared.

**Avoid when:**
- The data is small, fixed, or short-lived — a plain `Vec<u8>`, `&[u8]`, or `Arc<[u8]>`
  is simpler and carries no vtable/refcount machinery.
- You retain many tiny slices of large buffers and cannot afford the retention cost.
- You need frequent in-place mutation of data that is also widely shared — the
  immutable/unique split will force copies.
- You are in a hard-real-time or allocation-free context that a fixed-capacity buffer
  (`arrayvec`) serves better.

## Alternatives

- rust-lang/rust — `Vec<u8>` / `&[u8]` / `Arc<[u8]>` from std; use when you need a single
  owned or shared buffer and do not need cheap zero-copy sub-slicing.
- servo/rust-smallvec — use when buffers are usually tiny and you want to avoid heap
  allocation, rather than reference-counted sharing.
- bluss/arrayvec — use when the maximum size is known at compile time and stack-allocated,
  allocation-free buffers are preferable.
- tokio-rs/tokio-util — built on `bytes`; use its `codec`/`BytesCodec` framing helpers
  when you want framed reads/writes rather than the raw buffer types.
- fitzgen/bumpalo — use when many buffers share a single lifetime and bulk arena
  deallocation fits better than per-value reference counting.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2015-01 | Started as part of the early Tokio/Mio ecosystem.[^1] |
| 0.4.x | 2017–2018 | Widely-used pre-`async`/`await` line; `Buf`/`BufMut` established. |
| 0.5.0 | 2019-12 | Reworked alongside `std::future`; API cleanup ahead of 1.0.[^3] |
| 1.0.0 | 2021-01 | Stable API commitment; semver-stable 1.x line begins.[^3] |
| 1.x | 2021–2026 | Incremental additions (e.g. `no_std`, `extra-platforms`, `serde`), no breaking changes; last push 2026-07.[^1] |

## References

[^1]: tokio-rs/bytes repository metadata (stars, forks, created/pushed dates, MIT license), fetched via GitHub API 2026-07. https://github.com/tokio-rs/bytes
[^2]: bytes README — `no_std`, `extra-platforms` (portable-atomic), and `serde` feature notes and their MSRV implications. https://github.com/tokio-rs/bytes/blob/master/README.md
[^3]: bytes CHANGELOG — release history and semver-stability of the 1.x line. https://github.com/tokio-rs/bytes/blob/master/CHANGELOG.md

## Tags

rust, bytes, buffer, zero-copy, networking, io, tokio, no-std, reference-counting, memory-management
