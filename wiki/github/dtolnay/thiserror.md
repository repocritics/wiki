# dtolnay/thiserror

> A derive macro that generates `std::error::Error` implementations for your own struct and enum error types.

[GitHub repo](https://github.com/dtolnay/thiserror) ·
[docs.rs](https://docs.rs/thiserror) ·
[License: Apache-2.0 OR MIT](https://github.com/dtolnay/thiserror/blob/master/LICENSE-APACHE)

## Overview

thiserror is a procedural macro crate for Rust, authored and maintained by David Tolnay (`dtolnay`), first published in 2019[^1]. It removes the boilerplate of hand-implementing the standard library's `std::error::Error` trait: you annotate a struct or enum with `#[derive(Error)]` plus a few field/variant attributes, and the macro generates the `Display`, `Error::source`, and `From` impls for you.

The crate's defining design decision is that **it does not appear in your public API**[^2]. The derive expands to exactly the code you would have written by hand — an ordinary `impl std::error::Error for YourType`. Switching a type from hand-written impls to thiserror, or the reverse, is not a breaking change for downstream users. This is what separates thiserror from framework-style error crates that introduce their own error trait or wrapper type that leaks into signatures.

The most common point of confusion is thiserror versus anyhow, the author's companion crate. The rule of thumb from the README: use thiserror when you are designing a **library** and want callers to receive a specific, matchable error type; use anyhow when you are writing an **application** and just want a single easy error type to bubble up[^3]. They are frequently used together — libraries define thiserror enums, the application that consumes them collapses everything into `anyhow::Error`.

## Getting Started

```toml
[dependencies]
thiserror = "2"
```

```rust
use std::io;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader { expected: String, found: String },
    #[error("unknown data store error")]
    Unknown,
}
```

The `#[error("...")]` string becomes the `Display` impl; `{0}` / `{expected}` interpolate tuple and named fields directly. `#[from]` on a field generates a `From<io::Error>` impl (so `?` converts automatically) and marks that field as the error `source()`.

## Architecture / How It Works

thiserror is a compile-time-only proc-macro. `#[derive(Error)]` parses your type with `syn`, inspects the attributes, and emits trait impls with `quote`. Nothing from thiserror is present at runtime — the generated code depends only on `core`/`std`. Key attributes:

- **`#[error("msg")]`** drives the generated `Display`. It supports Rust format syntax plus shorthands: `{var}` / `{0}` reference fields, and trailing expression args can call functions on fields via `.var` / `.0` (e.g. `#[error("bad {:?}", first_char(.0))]`).
- **`#[from]`** generates a `From` conversion and implies `#[source]`. The variant must hold only the source (plus an optional backtrace); you cannot attach extra context fields to a `#[from]` variant.
- **`#[source]`** (or a field simply named `source`) is returned by `Error::source()`, forming the error chain.
- **`#[error(transparent)]`** forwards both `Display` and `source()` to the inner error without adding a message — used for "catch-all" variants and for opaque wrapper types that hide a private inner representation from the public API.
- **`#[backtrace]`** wires a `std::backtrace::Backtrace` field into the trait's `provide()` method. This path requires a nightly toolchain (Rust 1.73+) because `Error::provide` / backtrace provision are unstable[^2].

Because the macro leans on `syn` and `quote`, thiserror pulls the standard proc-macro dependency stack into your build graph. This is the crate's real cost: not runtime overhead (there is none), but compile time and the transitive `syn`/`proc-macro2`/`quote` dependency that most non-trivial Rust projects already carry anyway.

## Production Notes

- **Compile-time cost, not runtime cost.** The generated impls are what you'd write by hand, so there is no dispatch overhead. The tax is the proc-macro toolchain at build time. In practice `syn` is already in nearly every dependency tree, so the marginal cost is small — but for a genuinely dependency-minimal crate, hand-writing the `Error` impl avoids the proc-macro pull-in entirely.
- **`#[from]` and `?` collisions.** Because `#[from]` generates a blanket `From` impl per variant, you cannot have two `#[from]` variants that convert from the same source type — it produces conflicting `From` impls that fail to compile. Distinct source types are required.
- **`#[from]` cannot carry context.** A common frustration: you want `#[from]` ergonomics but also want to attach, say, the file path that failed. thiserror forbids extra fields on a `#[from]` variant. The workaround is a manual `From` impl or an explicit `.map_err(...)` at the call site.
- **Backtraces need nightly.** `#[backtrace]` and backtrace provision depend on unstable `Error::provide`; on stable Rust you can store a `Backtrace` field but cannot expose it through the trait. Do not design a stable-toolchain API around it.
- **no_std.** thiserror 2 added support for `no_std` targets by using `core::error::Error` (stabilized in Rust 1.81), so error types can derive `Error` without `std`[^4]. Verify your MSRV: enabling this path raises the minimum compiler version.
- **Not a replacement for error strategy.** thiserror only generates impls; it does not decide your error taxonomy. Over-deriving a giant enum with dozens of `#[from]` variants pushes conversion complexity onto callers and is a frequent design smell — sometimes a `transparent` opaque wrapper or a smaller hand-curated enum is better.

## When to Use / When Not

**Use when:**
- You are writing a **library** and want callers to pattern-match specific, documented error variants.
- You want the ergonomics of derived `Display`/`From`/`source` without those impls leaking a foreign type into your public API.
- You need a stable, matchable error contract for your crate's consumers.

**Avoid when:**
- You are writing an **application** and just want errors to bubble up with context — reach for anyhow (or eyre) instead.
- You must minimize build dependencies and compile time in a tiny crate — a hand-written `impl Error` avoids the proc-macro stack.
- You need rich diagnostic reports with source spans and help text — see miette.

## Alternatives

- dtolnay/anyhow — same author; a single dynamic error type for application code, the intended companion rather than a competitor.
- shepmaster/snafu — use when you want per-call-site context selectors and richer context attachment than `#[from]` allows.
- yaahc/eyre — use when you want anyhow-style dynamic errors but customizable report handlers (color, hooks).
- zkat/miette — use when you're building a compiler/CLI and want fancy diagnostic output with source spans and labels.
- yaahc/displaydoc — use when you only need a `Display` derive from doc comments and don't need `From`/`source` generation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-10 | Initial release; derive for `std::error::Error`[^1]. |
| 1.0.x | 2019–2024 | Long stable series; incremental attribute and format-arg refinements. |
| 2.0.0 | 2024-11 | Major release; `no_std` support via `core::error::Error`, format-arg improvements[^4]. |

## References

[^1]: thiserror on crates.io — version history and first publish. https://crates.io/crates/thiserror/versions
[^2]: thiserror README — attributes, `transparent`, backtrace, and API-invisibility notes. https://github.com/dtolnay/thiserror
[^3]: thiserror README, "Comparison to anyhow". https://github.com/dtolnay/thiserror#comparison-to-anyhow
[^4]: thiserror 2.0 release notes. https://github.com/dtolnay/thiserror/releases/tag/2.0.0

## Tags

rust, error-handling, derive-macro, proc-macro, error-trait, library, cargo-crate, dtolnay, no-std, std-error
