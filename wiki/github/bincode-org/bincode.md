# bincode-org/bincode

> A compact binary serialization codec for Rust — minimal encoding overhead, deliberately not self-describing, and the long-time default answer to "how do I turn this struct into bytes."

[GitHub repo (archived)](https://github.com/bincode-org/bincode) ·
[sourcehut (current home)](https://sr.ht/~stygianentity/bincode/) ·
[docs.rs](https://docs.rs/bincode) ·
License: MIT[^1]

## Overview

Bincode is a binary encoder/decoder for Rust values, started in 2014 in the earliest days of the Rust ecosystem and adopted early by the Servo project for inter-process communication. Its design premise is the opposite of JSON or CBOR: the format carries no field names, no type tags, and no schema — just the raw values in a defined order. Both sides must agree on the exact Rust types out of band. In exchange, output is close to the theoretical minimum size and encode/decode is little more than memory traffic. With roughly 275 million cumulative downloads on crates.io[^1] against ~3,100 GitHub stars, it is classic invisible infrastructure: far more used than starred.

The project's defining tradeoff is that "not self-describing" cuts both ways. It is what makes bincode small and fast, and also what makes it fragile: reorder two struct fields, change an integer width, or add a variant, and old bytes decode into garbage or errors with no format-level detection. Bincode is a wire and cache format, not an archival format.

The project's governance is currently unusual. In 2025 the maintainers moved primary development to sourcehut and archived the GitHub mirror, stating objections to GitHub's integration of generative AI[^2]. The GitHub repo (last push August 2025) is frozen at the 2.x line with 23 open issues preserved; version 3.0.0 was developed and released from sourcehut in December 2025[^1]. The crate is actively maintained — but issue history and current source now live in different places, which complicates archaeology.

## Getting Started

```bash
cargo add bincode
```

Since 2.0, bincode ships its own `Encode`/`Decode` derive macros and no longer requires serde[^3]:

```rust
use bincode::{Decode, Encode};

#[derive(Encode, Decode, PartialEq, Debug)]
struct Entity {
    id: u64,
    x: f32,
    y: f32,
}

fn main() {
    let config = bincode::config::standard();
    let entity = Entity { id: 7, x: 1.0, y: 2.0 };

    let bytes: Vec<u8> = bincode::encode_to_vec(&entity, config).unwrap();
    let (decoded, _len): (Entity, usize) =
        bincode::decode_from_slice(&bytes, config).unwrap();

    assert_eq!(entity, decoded);
}
```

Serde interop is behind the `serde` feature (`bincode::serde::encode_to_vec` etc.) for types that only derive `Serialize`/`Deserialize`.

## Architecture / How It Works

Bincode is a direct mapping from Rust's data model to bytes, governed by a written specification[^4]:

- **Integers** — under the 2.x `standard()` config, encoded as variable-length integers (small values take one byte). Fixed-width little-endian encoding is available via `config::legacy()` or explicit config, matching the 1.x default. This means **1.x and 2.x default output are not wire-compatible**; migrating a stored corpus requires decoding with `legacy()` config[^3].
- **Structs** — fields in declaration order, back to back. No names, no offsets, no padding.
- **Enums** — a variant index followed by the variant's payload. Adding, removing, or reordering variants changes indices.
- **Collections and strings** — a length prefix followed by elements. Strings are length-prefixed UTF-8.
- **`Option<T>`** — a one-byte tag then the value.

Because the format has no type information, `deserialize_any` is unsupported. Serde constructs that require self-description — `#[serde(flatten)]`, untagged enums, arbitrary-key maps into dynamic types — fail at runtime with bincode, a recurring source of confused bug reports across the ecosystem.

The 2.0 rewrite (released 2025 after a three-year release-candidate period) replaced the serde-only core with bincode's own `Encode`/`Decode` traits, made serde an optional integration, added `no_std` + `alloc` support, and rebuilt configuration as a compile-time typestate (`config::standard().with_big_endian()...`) so config choices are monomorphized rather than branched at runtime[^3].

## Production Notes

- **Schema evolution does not exist.** There is no field skipping, no defaults for missing fields, no version negotiation. Any persisted bincode data is implicitly coupled to a specific version of your Rust type definitions. Teams that persist bincode long-term either freeze the types, hand-roll versioned envelopes (`enum V { V1(...), V2(...) }`), or eventually migrate to a schema'd format. Budget for this on day one.
- **Untrusted input needs limits.** Length prefixes come from the wire; a hostile 8-byte prefix can request a multi-gigabyte allocation. Always set a size cap (`config::standard().with_limit::<N>()` in 2.x; `Options::with_limit` in 1.x) when decoding anything you did not produce. Decoding untrusted data without a limit is the classic bincode DoS.
- **The 1.x → 2.x migration is a real project, not a bump.** Different default int encoding (fixed → varint), different API surface (`serialize`/`deserialize` → `encode_to_vec`/`decode_from_slice`), and serde support moved behind a feature flag. Much of the ecosystem stayed on 1.3.3 (final 1.x release, April 2021) for years precisely because of stored-data compatibility[^1].
- **Floats are bitwise.** `NaN` payloads round-trip exactly, but nothing protects you from cross-language consumers making different float assumptions — bincode is effectively Rust-to-Rust. There are no maintained first-party implementations for other languages.
- **Not zero-copy.** Decoding allocates and copies (`String`, `Vec`). If your hot path is "read a large buffer and access a few fields," rkyv's access-in-place model beats bincode structurally, not by constant factors.
- **Issue tracker split.** Pre-2025 discussion is on the archived GitHub repo; current development is on sourcehut. Search both when debugging.

## When to Use / When Not

**Use when:**
- Both endpoints are Rust and compiled from the same (or contractually synced) type definitions — IPC, task queues, actor messages.
- You need a compact cache/snapshot format where you control invalidation (e.g., rebuildable caches keyed by app version).
- You want minimal dependency weight — bincode 2.x with derive works in `no_std` environments.
- Wire size and encode/decode cost matter more than debuggability of the bytes.

**Avoid when:**
- Data outlives the code that wrote it (user documents, long-term storage) — schema evolution is unsupported by design.
- Consumers exist in other languages — use a cross-language IDL format instead.
- You need to inspect payloads in flight — the format is opaque without the exact Rust types.
- You rely on serde's self-describing features (`flatten`, untagged enums).

## Alternatives

- jamesmunns/postcard — serde-based, `no_std`-first, even more size-frugal; the embedded community's default. Use it instead when targeting microcontrollers or when you want a stable spec'd format with a smaller footprint.
- rkyv/rkyv — zero-copy archival: access serialized data in place without a decode step. Use it instead when deserialization cost dominates and you can accept its more invasive derive model.
- 3Hren/msgpack-rust — MessagePack (rmp-serde): self-describing and cross-language at a modest size cost. Use it instead when non-Rust consumers or field-name resilience matter.
- tokio-rs/prost — Protocol Buffers: explicit schemas, forward/backward compatibility, cross-language. Use it instead when data or APIs must evolve across versions and teams.
- enarx/ciborium — CBOR: standardized (RFC 8949), self-describing. Use it instead when you need an IETF-standard format with serde ergonomics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2014-11 | First crates.io publish; repo created 2014-09[^1]. |
| 1.0.0 | 2018-02 | Stable serde-based API; fixed-width little-endian default. |
| 1.3.3 | 2021-04 | Final 1.x release; the ecosystem workhorse for the next four years[^1]. |
| 2.0.0-rc.1 | 2022-03 | Rewrite enters a release-candidate period that lasted three years[^1]. |
| 2.0.0 | 2025-03 | Own `Encode`/`Decode` traits, serde optional, `no_std`, varint default config[^3]. |
| — | 2025-08 | GitHub repo archived; primary development moves to sourcehut[^2]. |
| 3.0.0 | 2025-12 | First major release from sourcehut[^1]. |

## References

[^1]: crates.io — bincode (versions, dates, download counts). https://crates.io/crates/bincode
[^2]: bincode README migration notice (GitHub, archived). https://github.com/bincode-org/bincode
[^3]: bincode 2.x migration guide. https://github.com/bincode-org/bincode/blob/trunk/docs/migration_guide.md
[^4]: bincode format specification. https://github.com/bincode-org/bincode/blob/trunk/docs/spec.md
[^5]: bincode API documentation. https://docs.rs/bincode

## Tags

rust, serialization, binary-format, serde, encoding, no-std, wire-format, ipc, library
