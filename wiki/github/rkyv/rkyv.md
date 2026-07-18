# rkyv/rkyv

> Zero-copy deserialization for Rust — the archived bytes *are* the data
> structure, so "deserializing" is a pointer cast, not a parse.

[GitHub repo](https://github.com/rkyv/rkyv) ·
[Official website](https://rkyv.org) ·
[License: MIT](https://github.com/rkyv/rkyv/blob/master/LICENSE)

## Overview

rkyv (pronounced *archive*) is a Rust serialization framework built around one
idea: instead of encoding data into a format that must be parsed back into
native structures, write the native structure itself — with relative pointers
instead of absolute ones — so the resulting byte buffer can be memory-mapped or
sliced and used directly[^1]. Total deserialization is optional; accessing a
field of a multi-gigabyte archive costs a validation pass at most, not a full
decode. Started by David Koloski in November 2020, it has become the default
answer to "fastest Rust serialization" in the community, with ~4.3k stars and
active maintenance (last push July 2026).

The defining tradeoff is that rkyv is not an interchange format. There is no
schema, no self-description, and no cross-language story: the byte layout is
derived from your Rust type definitions plus rkyv's own version and feature
flags. What you gain in raw speed you pay for in coupling — the reader must
compile essentially the same types (and rkyv configuration) the writer did.
Compare-and-contrast: serde is a data-model abstraction over many formats;
rkyv is one format, in exchange for eliminating the decode step entirely.

The headline numbers come from the rust_serialization_benchmark, which shows
rkyv at or near the top for access latency[^2]. Worth knowing: that benchmark
is maintained by rkyv's own author. It is methodologically open and widely
cited, but it is not an independent shootout.

## Getting Started

```bash
cargo add rkyv
```

```rust
use rkyv::{deserialize, rancor::Error, Archive, Deserialize, Serialize};

#[derive(Archive, Deserialize, Serialize, Debug, PartialEq)]
#[rkyv(compare(PartialEq), derive(Debug))]
struct Test {
    int: u8,
    string: String,
    option: Option<Vec<i32>>,
}

fn main() {
    let value = Test {
        int: 42,
        string: "hello world".to_string(),
        option: Some(vec![1, 2, 3, 4]),
    };

    let bytes = rkyv::to_bytes::<Error>(&value).unwrap();

    // Validated zero-copy access (bytecheck feature, on by default)
    let archived = rkyv::access::<ArchivedTest, Error>(&bytes[..]).unwrap();
    assert_eq!(archived, &value);

    // Full deserialization back to the native type, when you need it
    let deserialized = deserialize::<Test, Error>(archived).unwrap();
    assert_eq!(deserialized, value);
}
```

Note the `#[rkyv(...)]` attribute syntax is the 0.8 API; 0.7 code uses
`#[archive(...)]` and is not source-compatible[^3].

## Architecture / How It Works

The `Archive` derive generates a parallel type for every annotated struct:
`Test` gets `ArchivedTest`, `String` maps to `ArchivedString`, `Vec<T>` to
`ArchivedVec<Archived<T>>`, and so on. Archived types have a stable,
explicitly-controlled memory layout and use **relative pointers** for
indirection — an offset from the pointer's own position rather than an
absolute address — which makes archives position-independent: the buffer can
be mapped at any address, on disk or over shared memory, and just work[^1].

Serialization is two-phase. First a `serialize` pass writes out-of-line data
(string bytes, vec contents) to the writer and produces a *resolver* capturing
where everything landed; then a `resolve` pass writes the fixed-size archived
value in place, encoding the relative offsets. Shared pointers (`Rc`/`Arc`)
are deduplicated during serialization.

Reading has three tiers: `access_unchecked` (unsafe pointer cast, no
validation), `access` (validates the buffer via the sister crate
**bytecheck**[^4] — checks offsets in-bounds, enums have valid discriminants,
UTF-8 is valid — then casts), and full `deserialize` back to native types.
Error handling across the API runs through **rancor**, and endian-agnostic
integer types come from **rend**; both are sister crates under the rkyv
org[^4]. Feature flags pin the wire format: `little_endian`/`big_endian` and
`pointer_width_{16,32,64}` must match between writer and reader. In-place
mutation of archived data is possible but deliberately restricted through the
`Seal` wrapper (0.8's replacement for 0.7's `Pin`-based API). `no_std` is
supported.

## Production Notes

**This is a cache format, not a storage format.** The layout depends on your
type definitions, rkyv's version, and feature flags. Renaming is fine; adding,
removing, or reordering fields silently changes the layout, and there is no
built-in schema evolution or version negotiation. The safe pattern is
ephemeral data you can regenerate — caches, indexes, IPC, snapshots — with a
version tag you bump to invalidate old archives. Wasmer, for example, uses
rkyv to cache compiled module artifacts, a canonical fit[^5]. Long-lived user
data across application versions is the wrong fit without serious discipline.

**Validation is where the zero-copy math gets honest.** `access` with
bytecheck walks the whole reachable structure — for untrusted input that is
an O(data) pass, which erodes the advantage over a fast decode-style format.
`access_unchecked` restores full speed but is `unsafe` in the strict sense:
a malformed or hostile buffer is out-of-bounds reads and type confusion.
Never point it at bytes you did not produce.

**Alignment bites.** Archived types have alignment requirements; a `&[u8]`
sliced out of a network buffer at an arbitrary offset will fail or UB. Use
`AlignedVec` (what `to_bytes` returns), align your mmap regions, or accept
the layout cost of the unaligned feature.

**Pre-1.0 churn is real.** 0.7 was the stable line for roughly three years;
0.8 (2024) reworked attributes, error handling (rancor), and the safe API, and
migration is mechanical but nontrivial[^3]. Third-party crates providing
`Archive` impls split across the 0.7/0.8 boundary — check your dependency
tree before upgrading. There is still no 1.0.

**Ergonomics tax.** Code that operates on archived data is written against
`ArchivedFoo` types, not `Foo` — different string type, different map types,
no `serde_json`-style "it's just my struct". Derive options
(`compare(PartialEq)`, passthrough `derive(...)`) narrow the gap but do not
close it. Budget for a real API boundary between archived and native worlds.

## When to Use / When Not

**Use when:**
- You memory-map large read-mostly data: search indexes, asset packs, game
  saves, compiled-artifact caches.
- Deserialization is your measured hotspot and both endpoints are Rust code
  you compile and deploy together.
- You need partial access to large payloads without decoding the whole thing.
- IPC or shared-memory transfer between trusted Rust processes.

**Avoid when:**
- You need cross-language interop or a public wire format — rkyv's layout is
  an implementation detail, not a spec.
- Data outlives the code that wrote it (config files, user documents,
  database-of-record) and you can't force regeneration.
- Input is untrusted and large — mandatory validation erases much of the win.
- Your payloads are small; serde + bincode/postcard is simpler and fast
  enough, with a vastly larger ecosystem.

## Alternatives

- serde-rs/serde + bincode-org/bincode — use instead when you want the
  ecosystem-standard data model, schema flexibility, and "fast enough" beats
  "zero-copy" plus its constraints.
- google/flatbuffers — use instead when you need zero-copy access *and*
  cross-language support with a schema/IDL and an evolution story.
- capnproto/capnproto-rust — use instead for zero-copy with schemas plus an
  RPC system; more infrastructure, more interop.
- jamesmunns/postcard — use instead for compact serde-based wire encoding on
  embedded/`no_std` targets where size matters more than decode time.
- frankmcsherry/abomonation — the historical predecessor to this approach;
  effectively unmaintained and soundness-questioned, listed for context, not
  adoption.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2020-11 | Initial release[^6]. |
| 0.7 | 2021 | Long-lived stable line (~3 years); `#[archive(...)]` attributes, `Pin`-based mutation. |
| 0.8 | 2024-05 | Major rework: unified `#[rkyv(...)]` attributes, rancor error handling, `Seal` mutation, safe-API overhaul[^3]. |

## References

[^1]: rkyv book — motivation, relative pointers, architecture. https://rkyv.github.io/rkyv/
[^2]: djkoloski/rust_serialization_benchmark — Rust serialization shootout including zero-copy benchmarks. https://github.com/djkoloski/rust_serialization_benchmark
[^3]: rkyv 0.8 release notes / migration. https://github.com/rkyv/rkyv/releases
[^4]: Sister crates: bytecheck (validation), rancor (errors), rend (endianness), ptr_meta. https://github.com/rkyv
[^5]: Wasmer engine artifact serialization uses rkyv. https://github.com/wasmerio/wasmer
[^6]: Repository created 2020-11-05 (GitHub API). https://github.com/rkyv/rkyv

## Tags

rust, serialization, zero-copy, deserialization, derive-macro, memory-mapping, no-std, performance, caching, data-format
