# dtolnay/anyhow

> A trait-object error type for Rust applications — one `anyhow::Error` absorbs any `std::error::Error`, carries context and a backtrace, and keeps `?` ergonomic.

[GitHub repo](https://github.com/dtolnay/anyhow) ·
[docs.rs](https://docs.rs/anyhow) ·
[crates.io](https://crates.io/crates/anyhow) ·
[License: Apache-2.0 OR MIT](https://github.com/dtolnay/anyhow#license)

## Overview

anyhow is David Tolnay's error-handling crate for Rust application code, first published in October 2019[^1]. It provides a single concrete type, `anyhow::Error`, that wraps any error implementing the standard `std::error::Error` trait as a trait object. The goal is narrow and deliberate: let application code propagate heterogeneous errors through `?` without hand-writing an error enum, while still attaching human-readable context and, where the compiler supports it, a captured backtrace.

The defining tension is application vs. library. anyhow is explicitly for the top of the stack, where you do not care what error type a function returns — you just want failure to be easy to bubble up and easy to read. Its companion crate thiserror (same author) sits at the other pole: libraries that want to expose a dedicated, matchable error type per API surface. Reaching for anyhow inside a published library is the most common misuse — callers lose the ability to match on specific variants because everything is erased to a trait object.

anyhow has sat at a stable `1.0` since almost its first release and is one of the most-depended-upon crates in the Rust ecosystem. At roughly 6.6k stars and 214 forks it under-indexes on GitHub relative to its actual reach, because the crate is small, finished by design, and consumed transitively far more than it is starred. Development is ongoing (last pushed June 2026) but consists of maintenance, new-compiler support, and no-std polish rather than API churn.

## Getting Started

```toml
[dependencies]
anyhow = "1.0"
```

```rust
use anyhow::{Context, Result};

fn get_cluster_info() -> Result<ClusterMap> {
    // `?` converts any std::error::Error into anyhow::Error
    let config = std::fs::read_to_string("cluster.json")
        .context("failed to read cluster.json")?;
    let map: ClusterMap = serde_json::from_str(&config)
        .context("cluster.json is not valid JSON")?;
    Ok(map)
}
```

A failure prints the context chain, root cause last:

```console
Error: failed to read cluster.json

Caused by:
    No such file or directory (os error 2)
```

One-off errors use the `anyhow!` macro; `bail!` is the early-return shorthand:

```rust
use anyhow::{anyhow, bail};

if !token.is_valid() {
    bail!("invalid token for user {}", user_id);
}
let n = map.get("count").ok_or_else(|| anyhow!("missing count"))?;
```

## Architecture / How It Works

`anyhow::Error` is a thin wrapper around a single owned pointer — not `Box<dyn Error>` but a custom narrow (thin, one-word) pointer to a heap allocation that co-locates a vtable, the error value, and optional backtrace. This is why `anyhow::Error` is one word wide where `Box<dyn Error + Send + Sync>` is two: anyhow stores the vtable in the allocation rather than the pointer, keeping `Result<T, anyhow::Error>` cheap to move.

Context is modeled as chaining. `.context("...")` and `.with_context(|| ...)` wrap the current error in a new node whose source is the previous error, building a linked list you can walk with `Error::chain()` or collapse to `Error::root_cause()`. Because every layer is still a real `std::error::Error`, the context you attach becomes part of the standard error source chain, not an anyhow-only sidecar.

Downcasting recovers the original typed error: `downcast_ref::<MyError>()`, `downcast_mut`, and `downcast` (by value) let recovery code branch on a specific cause after it has been erased. This is the escape hatch that makes trait-object erasure tolerable — you can still pattern-match a known cause near the top of the stack.

Backtraces are captured automatically on Rust 1.65 and newer, where `std::backtrace::Backtrace` stabilized[^2]. anyhow only captures its own backtrace if the wrapped error does not already provide one, and capture is gated by `RUST_BACKTRACE` / `RUST_LIB_BACKTRACE` at runtime, so the cost is opt-in. On older compilers no backtrace is captured.

The crate is `no_std`-capable: disabling the default `std` feature keeps almost the entire API, requiring only a global allocator[^3]. anyhow deliberately ships no derive macro — it holds no opinion about your typed errors and defers that entirely to thiserror or a hand-written impl.

## Production Notes

- **Do not use it in library public APIs.** Returning `anyhow::Error` from a library forces every downstream caller to `downcast` guesswork instead of matching typed variants. Use thiserror for library error types; keep anyhow for binaries, tests, build scripts, and glue.
- **Context strings are your logs.** The value anyhow delivers in production is the `Caused by:` chain. Terse `.context()` at each fallible boundary turns "No such file or directory" into a traceable narrative. Skipping context reduces anyhow to a slightly nicer `Box<dyn Error>`.
- **Backtraces are runtime-gated and version-gated.** A build on a pre-1.65 toolchain silently captures no backtrace. Even on new toolchains, nothing is captured unless `RUST_BACKTRACE=1` (or `RUST_LIB_BACKTRACE=1`) is set in the process environment. Teams that expect backtraces in incident logs must set this explicitly in their runtime, not just in CI.
- **`downcast` is by concrete type, not by trait.** Recovery code must name the exact original error type. Refactors that change an inner error type will silently stop matching — there is no compiler error, the `downcast_ref` just returns `None`. Treat downcast sites as coupling points.
- **Semver stability is a feature.** anyhow has stayed on `1.x` for years with no planned `2.0`. Pinning `anyhow = "1"` is safe long-term; the crate is intentionally near-complete rather than evolving.
- **no_std has sharp edges on old compilers.** On Rust older than 1.81, no_std mode can require an extra `.map_err(Error::msg)` when converting foreign errors, because the conversion trait `?` relies on was std-only there[^3].

## When to Use / When Not

**Use when:**
- You are writing application code, a CLI, a service binary, tests, or build tooling and want `?` to just work across many error types.
- You want readable context chains and optional backtraces without designing an error enum.
- You need to occasionally recover a specific cause via `downcast`.

**Avoid when:**
- You are publishing a library and callers need to match on specific error variants — use thiserror instead.
- You need zero-allocation error handling on a hot path; anyhow heap-allocates per error.
- You want the caller to see a structured, exhaustively-matchable error type rather than an opaque trait object.

## Alternatives

- dtolnay/thiserror — the companion crate; use when you are a library defining your own typed, matchable error enum with a derive macro.
- eyre-rs/eyre — a fork of anyhow with pluggable report handlers; use when you want custom error-report formatting (e.g. color-eyre) or extra context sections.
- shepmaster/snafu — context-selector derive with typed errors; use when you want thiserror-style typing plus anyhow-style context in one crate.
- rust-lang/rust `Box<dyn Error + Send + Sync>` — the std-only baseline; use when you want no dependency and can forgo context chaining and backtraces.
- rust-lang-deprecated/failure — the predecessor anyhow replaced; do not use in new code, it predates the `std::error::Error` improvements from RFC 2504[^4].

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-10 | Initial release; trait-object error built on `std::error::Error` per RFC 2504[^1][^4]. |
| 1.0.x | 2020–2022 | `context`/`with_context`, `anyhow!`/`bail!`/`ensure!` macros, downcasting, no_std feature matured. |
| 1.0.x | 2022-11 | Automatic backtrace capture on Rust ≥ 1.65 once `std::backtrace` stabilized[^2]. |
| 1.0.x | 2024–2026 | no_std ergonomics on newer compilers; ongoing maintenance, still `1.x` with no planned `2.0`[^5]. |

## References

[^1]: anyhow on crates.io — version history and first publish (October 2019). https://crates.io/crates/anyhow/versions
[^2]: Rust 1.65 release notes — stabilization of `std::backtrace::Backtrace`. https://blog.rust-lang.org/2022/11/03/Rust-1.65.0.html
[^3]: anyhow README, "No-std support". https://github.com/dtolnay/anyhow#no-std-support
[^4]: RFC 2504 — "Fix the error trait" (`std::error::Error` improvements anyhow is built on). https://github.com/rust-lang/rfcs/blob/master/text/2504-fix-error.md
[^5]: anyhow API documentation. https://docs.rs/anyhow

## Tags

rust, error-handling, error-type, trait-object, backtrace, cli, application-code, dtolnay, no-std, result
