# uuid-rs/uuid

> The de facto standard UUID crate for Rust — generate and parse RFC 9562
> identifiers with a zero-dependency core and feature-gated everything else.

[GitHub repo](https://github.com/uuid-rs/uuid) ·
[crates.io](https://crates.io/crates/uuid) ·
License: MIT OR Apache-2.0

## Overview

`uuid` is the Rust ecosystem's default library for 128-bit universally unique
identifiers. It began life inside the pre-1.0 Rust standard library and was
extracted into its own crate in 2014 as the stdlib slimmed down before Rust
1.0[^1]; today it lives under the community `uuid-rs` org and is among the
most-downloaded crates on crates.io[^2]. The GitHub star count (1.2k) wildly
understates its footprint — nearly every Rust web service, database client, and
distributed system transitively depends on it.

The crate's defining design choice is aggressive feature-gating. The core type
— parse, format, compare, const-construct — has no mandatory dependencies and
works in `no_std`. Every generator (`v4` random, `v7` timestamp-sortable, `v3`/
`v5` hashed, `v1`/`v6` MAC+time, `v8` custom) and every integration (`serde`,
`zerocopy`, `borsh`, `arbitrary`) is opt-in. That keeps compile times and
dependency trees minimal for the common case, at the cost of a recurring
newcomer stumble: `Uuid::new_v4()` simply doesn't exist until you enable the
`v4` feature.

The crate spent eight years at 0.x before committing to a stable API: 1.0.0
shipped in April 2022[^3], and the 1.x line has held semver-stable since —
notable discipline for infrastructure this widely depended on.

## Getting Started

```toml
[dependencies]
uuid = { version = "1", features = ["v4", "v7"] }
```

```rust
use uuid::{uuid, Uuid};

// v4: random (needs the "v4" feature)
let id = Uuid::new_v4();

// v7: Unix-timestamp-prefixed, roughly sortable (needs "v7")
let sortable = Uuid::now_v7();

// const-parsed at compile time, no feature needed
const NIL_ADJACENT: Uuid = uuid!("67e55044-10b1-426f-9247-bb680e5fe0c8");

// runtime parsing, also feature-free
let parsed = Uuid::parse_str("67e55044-10b1-426f-9247-bb680e5fe0c8").unwrap();
assert_eq!(parsed, NIL_ADJACENT);
```

## Architecture / How It Works

`Uuid` is a `#[repr(transparent)]` wrapper over `[u8; 16]` — `Copy`, 16 bytes,
no heap, directly transmutable to its byte representation (which the
`zerocopy`/`bytemuck` features formalize). Parsing and the `uuid!` macro are
const-capable, so invalid literals fail at compile time.

Formatting goes through borrowed adapter types — `simple()`, `hyphenated()`,
`urn()`, `braced()` — that write into stack buffers without allocating. This is
why the crate works in `no_std`: nothing in the core path touches `alloc`.

Generation is where the dependency tree grows. `v4` pulls in `getrandom` for
OS-level entropy; the `fast-rng` feature swaps in a userspace PRNG (seeded from
the OS) via `rand` for higher throughput. `v3`/`v5` pull in MD5/SHA-1 hashing.
`v1`/`v6`/`v7` use a `Timestamp` abstraction plus a `Context` type that manages
the counter bits, so IDs generated within the same tick still order correctly.
`v6`, `v7`, and `v8` sat behind an unstable cfg flag while the successor to RFC
4122 was in draft, and were stabilized in 1.6.0 (November 2023)[^4]; the spec
itself landed as RFC 9562 in May 2024[^5].

The `serde` integration is format-aware: human-readable serializers (JSON) get
the hyphenated string; binary serializers get the raw 16 bytes.

## Production Notes

**Pick v7, not v4, for database primary keys.** v4's uniform randomness
scatters inserts across a B-tree index, causing page splits and poor cache
locality at scale. v7's millisecond timestamp prefix makes keys roughly
append-ordered — this is the main practical reason the new versions exist.
Note the tradeoff: v7 leaks creation time.

**UUIDs are identifiers, not secrets.** Even v4 from a CSPRNG should not serve
as a bearer token or capability URL without deliberate threat modeling. v1
additionally leaks a node ID (historically the MAC address) and timestamp.

**WebAssembly needs configuration.** On `wasm32-unknown-unknown`, `getrandom`
has no default entropy source, so random generation fails to compile until you
select a backend[^6]. The mechanism has churned across `getrandom` major
versions (cargo feature vs. `RUSTFLAGS` cfg), and this is the single most
common CI breakage reported against the crate. Check the current README for
the incantation matching your `getrandom` version.

**Feature unification surprises.** Because Cargo unifies features across a
dependency graph, `Uuid::new_v4()` may compile in your crate only because some
transitive dependency enabled `v4`. Declare the features you actually use, or
a dependency bump can silently break your build.

**The 0.8 → 1.0 upgrade** reorganized feature flags and error types; crates
pinned to 0.8 coexist with 1.x in one build but their `Uuid` types are
incompatible, which surfaces as baffling "expected `Uuid`, found `Uuid`" errors
when two dependencies straddle the boundary.

**Parsing throughput** is rarely the bottleneck, but if it is (log ingestion,
bulk ETL), the maintainers themselves point to `uuid-simd` for SIMD-accelerated
parse/format[^1].

## When to Use / When Not

**Use when:**
- You need standard, interoperable 128-bit IDs — this is the canonical crate,
  and `sqlx`, `diesel`, `tokio-postgres`, and most ORMs integrate with it.
- You want time-sortable keys (v7) or deterministic namespace IDs (v5).
- You're in `no_std`/embedded and only need parse/format/compare.

**Avoid when:**
- You want shorter, URL-friendly IDs and don't need a standard — 36 hex chars
  is bulky in URLs and logs.
- You need strictly monotonic sequence numbers — UUID v7 ordering is only
  millisecond-coarse across processes; use a database sequence.
- Human-facing codes (order numbers, coupons) — UUIDs are hostile to reading
  aloud and transcription.

## Alternatives

- nugine/uuid-simd — use instead when UUID parse/format throughput is an
  actual measured bottleneck; SIMD-accelerated, same `Uuid` type.
- dylanhart/ulid-rs — use ULIDs when you want lexicographically sortable IDs
  with a compact 26-character Crockford base32 text form.
- nikolay-govorov/nanoid — use when you want short, URL-safe random IDs and
  interoperability with the JavaScript nanoid ecosystem.
- huxi/rusty_ulid — alternative ULID implementation with monotonic generation
  support.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014 | Extracted from the pre-1.0 Rust standard library[^1]. |
| 1.0.0 | 2022-04 | First stable release after ~8 years of 0.x[^3]. |
| 1.6.0 | 2023-11 | v6/v7/v8 stabilized (previously behind an unstable cfg)[^4]. |
| — | 2024-05 | RFC 9562 published, obsoleting RFC 4122[^5]. |
| 1.24.0 | 2026 | Current release line; repo actively pushed as of 2026-07. |

## References

[^1]: `uuid` library documentation. https://docs.rs/uuid
[^2]: crates.io — `uuid` crate page (download statistics). https://crates.io/crates/uuid
[^3]: uuid 1.0.0 release, GitHub — 2022-04. https://github.com/uuid-rs/uuid/releases/tag/1.0.0
[^4]: uuid 1.6.0 release notes — 2023-11. https://github.com/uuid-rs/uuid/releases/tag/1.6.0
[^5]: RFC 9562: Universally Unique IDentifiers (UUIDs) — May 2024. https://www.ietf.org/rfc/rfc9562.html
[^6]: `getrandom` WebAssembly support documentation. https://docs.rs/getrandom

## Tags

rust, uuid, identifiers, no-std, serialization, distributed-systems, database-keys, rfc-9562, library, parsing
