# tokio-rs/prost

> A Protocol Buffers implementation for Rust that generates plain, derive-based structs instead of a reflection-heavy runtime.

[GitHub repo](https://github.com/tokio-rs/prost) ·
[Docs](https://docs.rs/prost/) ·
[License: Apache-2.0](https://github.com/tokio-rs/prost/blob/master/LICENSE)

## Overview

`prost` compiles `proto2` and `proto3` schemas into idiomatic Rust types. Its
defining choice is that a Protobuf message becomes an ordinary `#[derive(Message)]`
struct with public fields, rather than an opaque object accessed through
generated getters/setters and a descriptor runtime. Encoding and decoding are
driven by a derive macro (`prost-derive`) that reads `#[prost(...)]` field
attributes; there is no reflection or `FileDescriptor` machinery in the core
crate. This makes generated code readable and cheap to work with, at the cost of
dropping features that reflection-based implementations provide.

The project originated with Dan Burkert and now lives under the tokio-rs
organization, which makes it the default Protobuf layer for the Tokio ecosystem —
notably `tonic`, the most-used gRPC stack in Rust, generates its message types
with prost. It uses `bytes::{Buf, BufMut}` rather than `std::io::{Read, Write}`
for its I/O abstraction, which lines it up with the rest of the async Rust stack
and enables zero-copy-ish buffer handling.

The important caveat for adopters is governance, stated plainly in the README:
the project is passively maintained. The current maintainer is not adding
features and does not have time to review feature PRs; bug fixes and security
fixes are still handled[^1]. The maintainer explicitly expects the official
`protobuf` project's Rust library to eventually supersede prost[^2]. Treat prost
as stable-and-frozen infrastructure, not a growing framework.

## Getting Started

```toml
[dependencies]
prost = "0.14"
# Only if you use Protobuf well-known types (Timestamp, Duration, Any, ...):
prost-types = "0.14"

[build-dependencies]
prost-build = "0.14"
```

Code generation runs at build time via a `build.rs`. `protoc` must be installed
and on `PATH` — since `prost-build` 0.11 it is no longer bundled or compiled for
you[^3].

```rust
// build.rs
fn main() {
    prost_build::compile_protos(&["proto/items.proto"], &["proto/"]).unwrap();
}
```

```rust
// src/main.rs — include the generated module and round-trip a message
use prost::Message;

pub mod items {
    include!(concat!(env!("OUT_DIR"), "/snazzy.items.rs"));
}

fn main() {
    let shirt = items::Shirt { color: "blue".into(), size: 1 };
    let mut buf = Vec::new();
    shirt.encode(&mut buf).unwrap();
    let decoded = items::Shirt::decode(&buf[..]).unwrap();
    assert_eq!(shirt.color, decoded.color);
}
```

## Architecture / How It Works

prost is a small family of crates with distinct roles:

- **`prost`** — the runtime: the `Message` trait, `encode`/`decode`, varint and
  wire-format handling, and re-exports of the derive macros.
- **`prost-derive`** — the proc-macro that implements `Message`, `Enumeration`,
  and `Oneof` by reading `#[prost(...)]` attributes. Gated behind the `derive`
  feature (on by default; disable to cut compile time when you only need the
  runtime).
- **`prost-build`** — the codegen driver. It shells out to `protoc` to parse
  `.proto` files into a `FileDescriptorSet`, then walks that set to emit Rust
  source. This is where package-to-module mapping, boxing decisions, and
  attribute injection happen.
- **`prost-types`** — Rust versions of the Protobuf well-known types and the
  descriptor messages themselves.

The type mapping is deliberately literal and worth internalizing because it
surfaces in every downstream API. `proto3` scalar fields become bare `T` (a
missing value decodes to `T::default()`); message fields become `Option<T>`;
`repeated` becomes `Vec<T>`; `map` becomes `HashMap` (or `BTreeMap` in
`no_std`); `oneof` becomes a generated `enum` wrapped in `Option`. Proto enums do
**not** become a Rust enum field — the field is stored as `i32` (to preserve
unknown values per the Protobuf open-enum rule) with accessor methods that
convert to/from the generated enum type. Recursive message types are boxed
automatically to keep struct sizes finite.

Because there is no descriptor runtime, prost cannot answer "what fields does
this message have?" at run time. If you need reflection, dynamic messages, or
JSON mapping by field name, you layer a separate crate (prost-reflect) on top,
using the optional `file_descriptor_set_path` output from `prost-build`.

## Production Notes

- **`protoc` is a hard build dependency.** CI images, Docker builds, and
  contributor machines all need it installed, and a version skew between
  `protoc` and prost's expectations can produce confusing codegen errors. This is
  the single most common onboarding friction. Vendoring the generated `.rs`
  files (committing them and skipping `build.rs`) is a legitimate escape hatch.
- **Minor versions are breaking.** prost is pre-1.0; `0.11 → 0.12 → 0.13 → 0.14`
  each carried breaking changes. More painful, the generated code embeds
  references to the prost runtime, so your prost, `prost-types`, `tonic`, and any
  other prost-generated dependency must all agree on the same prost minor
  version. A mismatch produces trait-coherence errors that are hard to read.
  Upgrades in a large dependency tree are effectively coordinated, not
  incremental.
- **The `i32` enum representation is a footgun.** Reading an enum field gives you
  an `i32`; you must call the accessor or `try_from` to get the typed value, and
  an out-of-range value silently reads back as the enum default. Code that
  matches on raw integers will not catch unknown variants.
- **`proto3` presence is lossy by default.** A scalar field set to its default
  value is indistinguishable from an unset field on the wire. Use `optional` in
  the schema (which yields `Option<T>`) when you need to know whether a value was
  actually present.
- **MSRV moves.** prost runs a rolling minimum-supported-Rust-version policy of
  at least six months; the current MSRV is 1.85[^4]. Pin your toolchain if you
  support older compilers.
- **Recursion limit is fixed at 100** and cannot be configured except by
  enabling the `no-recursion-limit` feature to remove it entirely — relevant for
  deeply nested or adversarial inputs.
- **`no_std` works** with `default-features = false`, but you must switch map
  fields to `BTreeMap` via `config.btree_map(&["."])` in your build script.

## When to Use / When Not

**Use when:**
- You want plain Rust structs for Protobuf messages with minimal ceremony.
- You are already in the Tokio/tonic ecosystem — prost is the path of least
  resistance for gRPC message types.
- You value readable generated code you can inspect and reason about.
- You need `no_std` / `bytes`-based encoding for embedded or async-heavy code.

**Avoid when:**
- You need runtime reflection, dynamic message construction, or descriptor-driven
  Protobuf JSON — the core crate does not do these (reach for prost-reflect or
  the `protobuf` crate).
- You want a project under active feature development and responsive review —
  prost is passively maintained by design.
- You cannot guarantee `protoc` in every build environment and don't want to
  vendor generated code.
- You want a serialization format without an external schema compiler — a Serde
  format may fit better.

## Alternatives

- stepancheg/rust-protobuf — the `protobuf` crate; older, supports descriptors,
  reflection, and text/JSON formats. Use when you need reflection that prost omits.
- protocolbuffers/protobuf — Google's official implementation, which now includes
  a Rust surface. Use when it reaches parity and you want the upstream-blessed path.
- hyperium/tonic — gRPC framework built on top of prost. Not an alternative to
  prost so much as its main consumer; use when you need RPC, not just message types.
- neoeinstein/prost-reflect — a companion, not a replacement: adds dynamic
  messages and reflection over prost-generated types. Use alongside prost when
  you need descriptors.
- capnproto/capnproto-rust — Cap'n Proto instead of Protobuf; zero-copy reads and
  no parse step. Use when decode latency dominates and you can change wire format.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-06 | Initial release by Dan Burkert. |
| 0.6 | 2019-11 | `bytes` 0.5 / async-era buffer abstractions. |
| 0.11 | 2022-08 | Bundled `protoc` removed; `protoc` now a required external dependency[^3]. |
| 0.12 | 2023-10 | Breaking release; moved further under tokio-rs stewardship. |
| 0.13 | 2024-05 | Breaking changes; continued codegen and API refinement. |
| 0.14 | 2025 | Current line; MSRV 1.85, passively-maintained posture stated in README[^1]. |

## References

[^1]: prost README, "Contributing" — states passive maintenance and that only bug/security fixes are handled. https://github.com/tokio-rs/prost#contributing
[^2]: prost README notes the maintainer expects the official `protobuf` project's Rust library to supersede prost. https://github.com/protocolbuffers/protobuf/tree/main/rust
[^3]: prost README, "protoc" — from prost-build 0.11, `protoc` is required and no longer bundled. https://github.com/tokio-rs/prost#protoc
[^4]: prost README, "MSRV" — rolling minimum-supported-Rust-version policy of at least six months; current MSRV 1.85. https://github.com/tokio-rs/prost#msrv

## Tags

rust, protobuf, protocol-buffers, serialization, code-generation, grpc, tokio, derive-macros, no-std, wire-format
