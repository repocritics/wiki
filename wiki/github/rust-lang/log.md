# rust-lang/log

> Rust's standard logging facade — a lightweight API that libraries log against and binaries wire an implementation into.

[GitHub repo](https://github.com/rust-lang/log) ·
[Official docs](https://docs.rs/log) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/log/blob/master/README.md)

## Overview

`log` is not a logger. It is a *facade*: a small set of macros (`error!`, `warn!`, `info!`, `debug!`, `trace!`) and a `Log` trait, plus a single global slot into which exactly one logging implementation is installed at runtime[^1]. Libraries depend only on `log` and emit records; the final binary chooses who actually formats and writes them (`env_logger`, `log4rs`, `fern`, a `tracing` bridge, and so on). This decoupling is the whole point — a crate deep in your dependency tree can log usefully without imposing a logging framework, an output format, or a runtime on you.

The design is deliberately minimal and has barely changed in years. The entire `0.4` series (since early 2018) is one uninterrupted semver-compatible line, and the maintainers treat the minimum supported Rust version and the public API as near-frozen surfaces[^2]. That stability is why essentially the whole Rust library ecosystem can share `log` as a common denominator: a crate written against `log` 0.4.0 still links against the latest 0.4.x.

The defining tension is scope. `log` models a *record* — a level, a target string, a message, and (since the `kv` feature) some key-value pairs — but it has no concept of spans, no async context propagation, and no structured-diagnostics story beyond flat key-values. For synchronous applications and libraries this is exactly enough. For async services that need causal context across `.await` points, the ecosystem has largely moved to `tracing`, and the honest framing today is "`log` for libraries and simple apps, `tracing` when you need spans."

## Getting Started

```toml
[dependencies]
log = "0.4"
env_logger = "0.11"   # one implementation, chosen by the binary
```

In a library, log against the facade and nothing else:

```rust
use log::{info, warn};

pub fn connect(url: &str) -> Result<(), Error> {
    info!(target: "net", "connecting to {url}");
    if url.is_empty() {
        warn!("empty url, using default");
    }
    Ok(())
}
```

In the binary, install an implementation exactly once, early:

```rust
fn main() {
    env_logger::init();          // reads RUST_LOG, sets the global logger + max level
    log::info!("startup");       // now records actually go somewhere
}
```

Run with `RUST_LOG=debug cargo run`. Any log call that executes *before* `init` is silently discarded — there is no buffering.

## Architecture / How It Works

The crate is three pieces:

1. **Macros.** `info!(...)` expands to a cheap level check against the current max-level filter, and only if it passes does it build a `Record` and call the global logger. Argument expressions are not evaluated when the level is disabled, so `debug!("{}", expensive())` costs almost nothing in a release build with debug filtered out.
2. **The `Log` trait.** Three methods: `enabled(&Metadata)`, `log(&Record)`, and `flush()`. An implementation is any type that satisfies these. `Record` carries the level, target, message args, and (optionally) module path, file, line, and key-values; `Metadata` is the subset available for the fast `enabled` check.
3. **The global logger + level filter.** `set_logger` (or `set_boxed_logger`) installs the singleton behind an atomic pointer; it can only succeed once and returns `SetLoggerError` thereafter. A separate `set_max_level` controls the runtime `LevelFilter`. These are two independent knobs, which is a frequent source of bugs (below).

Two layers of filtering exist. **Runtime** filtering is `set_max_level` / `STATIC_MAX_LEVEL`, checked on every macro call. **Compile-time** filtering is done through cargo features (`max_level_*` and `release_max_level_*`): setting `release_max_level_info` makes `debug!`/`trace!` compile to nothing in release builds, so they impose zero runtime cost and cannot be re-enabled without recompiling[^3].

Structured logging lives behind the `kv` feature. A record can carry typed key-value pairs (`info!(id = 42, user:serde; "login")`), capturing values via `Debug`, `Display`, `serde`, `sval`, or error impls. This feature spent years as `kv_unstable` before being stabilized and renamed[^4]; consumers pinning old versions may still see the unstable name.

## Production Notes

**The silent-nothing failure mode.** After `set_logger`, the global max level still defaults to `Off`. A hand-rolled logger that installs itself but forgets `set_max_level(LevelFilter::Trace)` (or similar) produces no output at all, with no error. Every well-behaved implementation's `init()` sets both; the trap is only for people writing their own `Log`.

**One logger, forever.** The global slot is write-once. You cannot swap loggers at runtime through the standard API, and a second `set_logger` fails. This bites test suites (parallel tests fighting over the global) and plugin/reload scenarios. Libraries must never call `set_logger` — that decision belongs to the binary. Reconfigurable output requires an implementation that supports it internally (e.g. a reloadable handle) rather than replacing the logger.

**Compile-time filters are sticky and global to the build.** `release_max_level_*` is resolved per compilation, so a dependency compiled under a restrictive static max level will never emit finer records even if you raise the runtime level. This is invisible until someone wonders why `RUST_LOG=trace` shows nothing from a particular crate.

**Dynamic-library / FFI boundaries.** The global logger lives in one linked copy of `log`. Across a `cdylib`/plugin boundary the plugin has its own uninitialized `log` state, so records emitted inside a dynamically loaded library go nowhere unless you explicitly forward them across the boundary[^5].

**Performance.** The disabled path is a level compare plus an atomic load — negligible. The enabled path is a dynamic dispatch through the trait object plus whatever the implementation does (often lock contention on a shared writer, or blocking I/O to stderr). High-throughput code should filter aggressively and prefer implementations with buffering / non-blocking writers; `log` itself imposes no async or batching.

**Interop with `tracing`.** A codebase can bridge both directions: `tracing-log` captures `log` records into a `tracing` subscriber, and `tracing-subscriber` can consume `log` events. This lets a library keep using `log` while an application standardizes on `tracing`. Do not install two consumers of the same records or you will double-log.

## When to Use / When Not

**Use when:**
- You are writing a library and want to emit diagnostics without forcing a framework or runtime on downstream users.
- The application is synchronous or its context needs are flat (level + target + message + a few fields).
- You want the lowest-common-denominator that every Rust logging implementation already understands.
- You need `no_std` logging or aggressive compile-time stripping of verbose levels.

**Avoid when:**
- You need spans, causal context across async tasks, or structured diagnostics — reach for `tracing`.
- You want rich structured output as the primary model rather than an opt-in feature.
- You need runtime-swappable logging destinations as a first-class capability.

## Alternatives

- tokio-rs/tracing — use instead when you need spans, async-aware context propagation, or structured events as the default model; now the common choice for services.
- slog-rs/slog — use instead when you want structured logging built around explicit `Logger` values passed through your code rather than a global.
- knurling-rs/defmt — use instead on embedded `no_std` targets where deferred formatting and code size matter more than a familiar API.
- rust-cli/env_logger — not a facade replacement but the usual *implementation* to pair with `log` for CLIs and simple binaries.
- estk/log4rs — use as the implementation when you need file appenders, rotation, and external config, still logging against the `log` facade.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.3.x | ~2015 | Early facade, extracted from the pre-1.0 Rust standard distribution. |
| 0.4.0 | 2018-01 | Major rewrite: `Record`/`Metadata` builders, `no_std` support, the long-lived stable API[^2]. |
| kv_unstable | ~2020 | Structured key-value logging introduced behind an unstable feature. |
| 0.4.21 | 2024-03 | `kv` structured-logging feature stabilized (renamed from `kv_unstable`)[^4]. |

The `0.4` line has held semver compatibility since 2018; MSRV bumps are treated as significant and called out in release notes[^2].

## References

[^1]: `log` crate documentation. https://docs.rs/log
[^2]: rust-lang/log README and CHANGELOG — MSRV / stability policy. https://github.com/rust-lang/log/blob/master/CHANGELOG.md
[^3]: `log` docs, "Available logging implementations" and compile-time filter features (`max_level_*`, `release_max_level_*`). https://docs.rs/log/latest/log/#compile-time-filters
[^4]: `log` structured logging (`kv`) feature. https://docs.rs/log/latest/log/kv/index.html
[^5]: rust-lang/log issue #421 — FFI-safe logging across dynamic library boundaries. https://github.com/rust-lang/log/issues/421

## Tags

rust, logging, facade, library, observability, structured-logging, no-std, crates-io, diagnostics
