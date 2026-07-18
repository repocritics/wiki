# BurntSushi/byteorder

> Rust's original endianness crate — read and write big/little-endian numbers,
> with the byte order picked as a type parameter at compile time.

[GitHub repo](https://github.com/BurntSushi/byteorder) ·
[Documentation](https://docs.rs/byteorder) ·
[License: Unlicense OR MIT](https://github.com/BurntSushi/byteorder/blob/master/COPYING)

## Overview

byteorder is a small Rust library by Andrew Gallant (BurntSushi, also the
author of ripgrep and the `regex` crate) for encoding and decoding integers
and floats in explicit byte order. It predates Rust 1.0 — first published in
February 2015[^1] — and for the language's first four years it was *the* way
to do endian-aware I/O, since the standard library had nothing. It remains
one of the most-downloaded crates on crates.io: over 694 million downloads
against a modest 1,091 GitHub stars[^2], the classic signature of invisible
plumbing that half the ecosystem depends on transitively rather than a
project developers seek out.

The defining design choice is compile-time endianness: you write
`rdr.read_u32::<BigEndian>()`, and monomorphization turns each call into a
plain load plus (at most) a byte-swap instruction. There is no runtime
dispatch, no configuration object, no cost. The trade-off is that the crate's
own reason to exist has been shrinking since Rust 1.32 (January 2019) added
`to_le_bytes` / `from_le_bytes` to the primitive integer types — a fact the
README itself points out[^3]. byteorder today is finished software: the last
release (1.5.0) shipped in October 2023 and the last repository push was
September 2024[^2]. That is maintenance mode by choice, not abandonment —
the API surface is tiny, stable at 1.x since December 2016, and there is
essentially nothing left to add.

## Getting Started

```toml
[dependencies]
byteorder = "1"
```

```rust
use std::io::Cursor;
use byteorder::{BigEndian, LittleEndian, ReadBytesExt, WriteBytesExt};

// Reading: the type parameter selects the byte order.
let mut rdr = Cursor::new(vec![2, 5, 3, 0]);
assert_eq!(517, rdr.read_u16::<BigEndian>().unwrap());
assert_eq!(768, rdr.read_u16::<BigEndian>().unwrap());

// Writing.
let mut wtr = vec![];
wtr.write_u16::<LittleEndian>(517).unwrap();
assert_eq!(wtr, vec![5, 2]);
```

For `no_std` use, disable the default `std` feature:

```toml
[dependencies]
byteorder = { version = "1", default-features = false }
```

## Architecture / How It Works

The entire crate is three pieces:

1. **`ByteOrder` trait** — slice-based primitives (`read_u16`, `write_u64`,
   `read_f32`, bulk `read_u32_into`, variable-width `read_uint`, and so on).
   The trait is sealed: it cannot be implemented outside the crate[^3].
2. **`BigEndian` / `LittleEndian`** — uninhabited enums (no values can ever
   exist) used purely as type parameters. `NetworkEndian` is a type alias
   for `BigEndian`; `NativeEndian` aliases whichever matches the target.
3. **`ReadBytesExt` / `WriteBytesExt`** — extension traits with blanket
   impls for every `std::io::Read` / `Write`, adding `read_u16::<E>()`-style
   methods. These are gated behind the `std` feature because `io` traits do
   not exist in `core`.

Internally, reads copy bytes into an aligned local buffer (via
`copy_nonoverlapping`) and then apply the integer `to_be` / `to_le`
conversions, so unaligned input slices are handled correctly — unlike a
naive pointer cast, which is undefined behavior on unaligned data. After
monomorphization and inlining, a `read_u32::<LittleEndian>` on a
little-endian target compiles to a single unaligned load; on the "wrong"
endianness it adds one `bswap`. Floats are round-tripped through their bit
patterns. The bulk slice methods (`read_u32_into` etc.) convert whole arrays
and give the optimizer a vectorizable loop.

The coupling story is trivially good: no dependencies, no build script, no
macros. It is the kind of leaf crate that never causes a dependency conflict.

## Production Notes

- **You may not need it.** Since Rust 1.32,
  `u32::from_le_bytes(buf[0..4].try_into().unwrap())` covers fixed-size
  conversions with zero dependencies[^3]. byteorder's remaining value is the
  `Read`/`Write` extension methods (streaming parsers), bulk slice
  conversions, and variable-width `read_uint`/`write_uint`. For new code
  that only touches a few fields, prefer std.
- **Panics on short slices.** The `ByteOrder` slice methods (`LittleEndian::
  read_u32(&buf)`) panic if the slice is too short; only the `io`-based
  extension methods return `Result`. Parsing untrusted input with the slice
  API means you must bounds-check first or accept panics as your error path.
- **Endianness is static.** Because byte order is a type parameter, formats
  where a header flag decides endianness at runtime (e.g. TIFF, ELF) force
  you to branch into two monomorphized code paths or write generic helper
  functions — there is no `dyn ByteOrder` (the trait is sealed and the types
  uninhabited).
- **`no_std` loses half the API.** Without `std` you keep the `ByteOrder`
  slice methods but lose `ReadBytesExt`/`WriteBytesExt` entirely.
- **Maintenance profile.** Single-maintainer, but the surface is ~one file
  of well-tested code with a conservative MSRV policy (currently 1.60,
  bumpable only in minor releases)[^3]. Release cadence of years is a
  feature here, not a risk: 1.0 semver compatibility has held since 2016.
  Licensing (Unlicense OR MIT) is as permissive as licensing gets.

## When to Use / When Not

**Use when:**
- You are writing a streaming binary parser/serializer over `io::Read`/
  `io::Write` and want endian-explicit typed reads.
- You need bulk endian conversion of numeric slices.
- It is already in your dependency tree (it very likely is) and consistency
  beats novelty.

**Avoid when:**
- You only convert a handful of fixed-size fields — std's
  `from_le_bytes`/`to_le_bytes` does it dependency-free.
- You are mapping whole structs onto byte buffers — zerocopy-style safe
  transmute avoids per-field code entirely.
- You are parsing a complex format with headers, offsets, and variants — a
  declarative or combinator framework scales better than hand-rolled reads.

## Alternatives

- Rust standard library — `{integer}::from_le_bytes` / `to_be_bytes` (since
  1.32); use instead for fixed-size conversions with no dependency.
- tokio-rs/bytes — `Buf`/`BufMut` traits with `get_u16`/`put_u32_le`; use
  when you are already in the tokio/bytes buffer ecosystem.
- google/zerocopy — derive-based safe transmute between structs and byte
  slices; use when you want zero-copy views instead of per-field reads.
- jam1garner/binrw — declarative `#[derive(BinRead)]` for whole binary
  formats; use when the format is complex enough to deserve a schema.
- rust-bakery/nom — parser combinators; use when parsing variable-length,
  branching binary (or text) grammars.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-02-03 | Initial release, three months before Rust 1.0[^1]. |
| 1.0.0 | 2016-12-30 | API stabilized; 1.x compatibility unbroken since[^1]. |
| 1.3.0 | 2019-01-19 | 128-bit integer support[^1]. |
| 1.4.3 | 2021-03-10 | Last 1.4.x patch release[^1]. |
| 1.5.0 | 2023-10-06 | Current release; MSRV 1.60[^1][^3]. |

## References

[^1]: crates.io version history for byteorder. https://crates.io/crates/byteorder/versions
[^2]: GitHub repository metadata, fetched 2026-07-18 (1,091 stars, 159 forks, last push 2024-09-25); crates.io download count 694M+, fetched same day. https://github.com/BurntSushi/byteorder
[^3]: byteorder README — installation, `no_std`, MSRV policy, and the "Alternatives" note on Rust 1.32's `to_le_bytes`/`from_le_bytes`. https://github.com/BurntSushi/byteorder#readme

## Tags

rust, endianness, byte-order, binary-parsing, serialization, no-std, io, encoding, low-level, library
