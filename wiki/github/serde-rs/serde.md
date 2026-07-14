# serde-rs/serde

> Compile-time serialization framework for Rust: derive traits once, plug in any data format.

[GitHub repo](https://github.com/serde-rs/serde) ·
[Official website](https://serde.rs/) ·
License: Apache-2.0 OR MIT (dual)[^1]

## Overview

Serde is the de facto serialization layer for the Rust ecosystem. It splits the
problem into two halves that meet at a fixed intermediate abstraction called the
Serde data model: your types implement `Serialize` / `Deserialize` once, and each
wire format (JSON, YAML, CBOR, MessagePack, TOML, bincode, …) implements a
`Serializer` / `Deserializer` once. Any type then works with any format without
N×M glue code[^2]. It is maintained by David Tolnay (dtolnay), the author of
`syn`, `proc-macro2`, `thiserror`, and `anyhow`.

The defining design choice is that everything happens at compile time via the
`serde_derive` procedural macro and monomorphization — there is no runtime
reflection, no schema registry, and no dynamic type information. This is why Serde
is fast at runtime and why it is a notable contributor to Rust build times: the
generated `Visitor` code and the `syn`-based macro expansion are heavy. That
tradeoff — near-zero runtime cost paid for with compile time and binary size — is
the tension every serious Serde user eventually confronts.

Serde has held a stable `1.0` API since 2017 with no breaking release since, which
is a large part of why the ecosystem standardized on it. The cost of that
ubiquity is that Serde sits deep in almost every Rust dependency tree, so its
decisions have outsized blast radius (see the 2023 precompiled-binary episode
below).

## Getting Started

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"   # the format lives in a separate crate
```

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    let json = serde_json::to_string(&p).unwrap(); // {"x":1,"y":2}
    let back: Point = serde_json::from_str(&json).unwrap();
    println!("{back:?}");
}
```

## Architecture / How It Works

The core is the **data model**: 29 canonical types (bool, the integer/float
widths, char, string, byte array, option, unit, unit/newtype/tuple/struct
variants, seq, map, struct, …). `Serialize` maps a Rust value *into* that model;
a `Serializer` maps the model *out* to bytes. Deserialization runs in reverse
using the **visitor pattern**: `Deserialize` hands the `Deserializer` a `Visitor`,
and the format drives it — this is what lets Serde deserialize the same struct
from self-describing formats (JSON) and non-self-describing ones (bincode) without
the type knowing which.

`serde_derive` is a separate proc-macro crate (pulled in by the `derive` feature).
It expands `#[derive(Serialize, Deserialize)]` into hand-written-equivalent trait
impls at compile time. Field- and container-level `#[serde(...)]` attributes
(`rename`, `default`, `skip`, `flatten`, `tag`, `with`, `deserialize_with`,
`borrow`) steer that codegen.

Deserialization is parameterized by a lifetime, `Deserialize<'de>`, which is what
makes **zero-copy** deserialization possible: a field typed `&'de str` or
`&'de [u8]` borrows directly out of the input buffer instead of allocating, when
the format and input allow it (`#[serde(borrow)]`). This lifetime is also the
single biggest source of confusing compiler errors for newcomers.

Formats are not in this repo. `serde_json`, `bincode`, `ciborium` (CBOR),
`rmp-serde` (MessagePack), `toml`, `postcard`, and `ron` are independent crates
implementing the `Serializer`/`Deserializer` traits. Serde itself is `no_std`
capable (opt out of the `std` feature; `alloc` is a middle ground).

## Production Notes

- **Compile time and binary size.** `serde_derive` → `syn`/`quote` is one of the
  most-cited contributors to slow Rust builds, and monomorphization across many
  types inflates binary size. Mitigations: share generated code, avoid deriving on
  huge generated types, or use a lighter alternative (see below) where JSON-only.
- **The 2023 precompiled binary.** `serde_derive` 1.0.171 began shipping a
  precompiled macro binary to cut compile time; the community objected on
  supply-chain, auditability, and reproducible-build grounds, and it was reverted
  to build-from-source in 1.0.184[^3]. Worth knowing if you pin old `serde_derive`
  versions or vendor dependencies.
- **`#[serde(flatten)]` footguns.** Flatten buffers fields into an intermediate map,
  so it silently breaks or misbehaves with non-self-describing formats (bincode,
  postcard) and interacts badly with `deny_unknown_fields`, numeric key coercion,
  and `u128`/`i128`. Prefer explicit fields when the format is binary.
- **Untrusted input is not automatically safe.** Serde does not impose universal
  recursion or allocation limits; a hostile payload can trigger deep recursion or
  large pre-allocations depending on the format crate. `serde_json` caps recursion,
  but capacity/DoS hardening is the format's and your responsibility.
- **`serde_yaml` is unmaintained.** dtolnay archived it in 2024[^4]; new code
  should use a maintained fork (`serde_yml`, `serde_norway`) or another format.
- **MSRV split.** `serde` supports Rust 1.56; `serde_derive` requires Rust 1.71[^5].
  A workspace that only needs the runtime traits can hold an older toolchain than
  one that derives.

## When to Use / When Not

**Use when:**
- You need any Rust type to round-trip through one or more wire formats.
- You want compile-time-checked (de)serialization with no runtime reflection cost.
- You are writing a *format* crate and want instant compatibility with the ecosystem.
- You need `no_std` / embedded serialization (with a `no_std`-friendly format).

**Avoid / reconsider when:**
- Build time and binary size dominate and you only touch JSON — a minimal codec is lighter.
- You need zero-copy *access* to large archived data without a deserialize pass — an archival format fits better.
- You need cross-language schemas with versioning/evolution guarantees — a schema-first IDL is a better fit.
- You require deterministic, canonical byte output for hashing/consensus — pick a format built for that contract.

## Alternatives

- rkyv/rkyv — zero-copy deserialization via archived layouts; use when you must read large data structures without a full decode pass.
- dtolnay/miniserde — JSON-only, no monomorphization blowup; use when compile time and binary size matter and you only need JSON.
- not-fl3/nanoserde — near-zero dependencies, fast builds; use for small projects that want to avoid the `syn` compile cost.
- near/borsh-rs — deterministic binary encoding; use when you need canonical bytes for blockchains/consensus.
- tokio-rs/prost — Protobuf codegen; use when you need a versioned cross-language schema rather than Rust-native types.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2013-11-13 | Early pre-1.0 serialization experiments[^6]. |
| 1.0.0 | 2017-04 | Stable API; no breaking release since[^2]. |
| derive feature | 2017+ | `serde_derive` folded in behind the `derive` feature. |
| 1.0.171 | 2023-08 | Introduced precompiled `serde_derive` binary — reverted[^3]. |
| 1.0.184 | 2023-08 | Reverted to build-from-source after community pushback[^3]. |
| ongoing 1.0.x | 2026 | Still `1.x`; `serde_derive` MSRV raised to 1.71[^5]. |

## References

[^1]: Dual-licensed Apache-2.0 OR MIT per the repository README license section. GitHub's API surfaces a single SPDX id (`Apache-2.0`). https://github.com/serde-rs/serde#license
[^2]: Serde overview and data model. https://serde.rs/
[^3]: "serde_derive to compile a precompiled binary" discussion / revert. https://github.com/serde-rs/serde/issues/2538
[^4]: `serde_yaml` archival notice. https://github.com/dtolnay/serde-yaml
[^5]: MSRV badges (serde 1.56, serde_derive 1.71) in the project README. https://github.com/serde-rs/serde
[^6]: Repository metadata (created 2013-11-13), GitHub API `repos/serde-rs/serde`.

## Tags

rust, serialization, deserialization, serde, json, no-std, derive-macro, data-format, proc-macro, framework
