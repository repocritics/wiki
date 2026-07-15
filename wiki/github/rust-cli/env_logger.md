# rust-cli/env_logger

> The default concrete logger for Rust's `log` facade — filtering driven entirely by the `RUST_LOG` environment variable.

[GitHub repo](https://github.com/rust-cli/env_logger) ·
[docs.rs](https://docs.rs/env_logger) ·
[License: Apache-2.0](https://github.com/rust-cli/env_logger/blob/main/LICENSE-APACHE)

## Overview

`env_logger` is a logging *implementation* for the [`log`](https://docs.rs/log) crate — the standard logging facade in the Rust ecosystem. `log` defines the macros (`error!`, `warn!`, `info!`, `debug!`, `trace!`) and a global `Log` trait, but ships no output backend; `env_logger` is the backend that most binaries and examples reach for first. It is configured almost entirely through one environment variable, `RUST_LOG`, and writes human-readable lines to stderr. Like most crates in the Rust ecosystem it is dual-licensed MIT OR Apache-2.0 despite GitHub reporting a single SPDX id[^1].

The crate is one of the oldest pieces of Rust tooling — it predates Rust 1.0 and lived in the `rust-lang` org before being split into its own repository under the `rust-cli` working group in 2017[^2]. Its scope is deliberately narrow: it is meant for command-line programs and test binaries that want zero-config, grep-friendly logs, not for services that need structured events, spans, or high-throughput async logging.

The defining tension is exactly that narrowness. `env_logger` is trivial to add and requires no code to configure — a strength for CLIs and reproductions. But it installs a single global logger that can only be set once, holds a lock per log call, and offers no spans, no structured fields, and no file/rotation targets. Projects that outgrow "print filtered lines to stderr" almost always migrate to `tracing` rather than extending `env_logger`.

## Getting Started

```console
$ cargo add log env_logger
```

```rust
use log::{info, warn};

fn main() {
    env_logger::init();   // reads RUST_LOG; installs the global logger

    info!("starting up");
    warn!("cache miss for key {}", 42);
}
```

```bash
$ RUST_LOG=info ./main
[2026-07-15T06:09:06Z INFO  myapp] starting up
[2026-07-15T06:09:06Z WARN  myapp] cache miss for key 42
```

The filter string is per-module and comma-separated: `RUST_LOG=warn,myapp::db=debug,hyper=off`. In tests use `env_logger::builder().is_test(true).try_init()` so output is captured per test and a second init does not panic.

## Architecture / How It Works

`env_logger` sits on top of two layers it does not own. The `log` crate exposes a global `max_level` atomic that the logging macros check first — disabled levels are skipped with a cheap atomic load and never format their arguments. Only records that pass that gate reach `env_logger`'s `Log::log` implementation.

Per-module filtering is delegated to a companion crate, `env_filter`, which was extracted from `env_logger` so the same `RUST_LOG` grammar can be reused by other loggers[^3]. `env_filter` parses the directive string into an ordered set of `(module_path, level)` rules plus an optional `/regex` message filter, and answers `enabled()` for each record. Because this runs per record (after the coarse `max_level` gate), a very permissive `RUST_LOG` on a hot path still costs a rule walk per call.

Output and coloring go through the `anstyle`/`anstream` ecosystem. Version 0.11 replaced the older `atty` + `termcolor` + `humantime` dependency stack with `anstream` for terminal detection and ANSI handling, and made color, timestamp, and regex support opt-in cargo features[^4]. The switch away from `atty` was partly motivated by that crate's soundness advisory (RUSTSEC-2021-0145). Timestamps and the level/target layout are produced by a built-in formatter; the `Builder::format` closure lets you replace the whole line.

Configuration is the `Builder` type: `from_env` / `from_default_env` seed it from `RUST_LOG` (and `RUST_LOG_STYLE` for color), then `.filter()`, `.target()`, `.format()`, `.write_style()`, and `.is_test()` override pieces before `.init()` or `.try_init()` installs it via `log::set_logger`. `set_logger` is a one-shot for the whole process — this is the hard constraint the rest of the design follows from.

## Production Notes

- **One global logger, set once.** `init()` panics if a logger is already installed (common when a dependency also calls it, or across test runs). Prefer `try_init()` and handle the `Err`. There is no way to swap the logger later or scope it per subsystem.
- **Per-call lock.** Each record acquires a lock around the stderr writer. Under heavy multi-threaded logging this serializes threads and becomes a contention point. `env_logger` is not built for high-volume production log streams; it is built for CLIs and tests.
- **stderr/stdout only.** Targets are `Stderr` (default), `Stdout`, or a user-supplied `Target::Pipe`. There is no built-in file target, no rotation, and no async/non-blocking writer. File logging, rotation, or multiple sinks mean a different crate (`log4rs`, `flexi_logger`, `fern`) or `tracing`.
- **Output format is explicitly unstable.** The maintainers make no guarantee about the default line format across 0.x releases and warn against parsing it programmatically — use a custom `format` closure if you must machine-read the output.
- **Feature-gated defaults changed.** Since 0.11, color/timestamp/regex are optional features; a `default-features = false` dependency (or a minimal build) can silently drop timestamps or the `/regex` filter. Check the enabled features when output looks different after an upgrade.
- **Upgrade friction, 0.10 → 0.11.** The dependency overhaul (anstyle migration, `env_filter` split, feature reshuffle) bumped the MSRV and changed which features are on by default; it is the most disruptive recent jump. Most call sites (`init`, `builder`, `RUST_LOG`) stayed source-compatible.
- **Test interleaving.** With `is_test(true)`, output is captured, but parallel tests still interleave unless you set `RUST_TEST_THREADS=1` or run a single test.

## When to Use / When Not

**Use when:**
- You are writing a CLI, example, or reproduction and want filtered stderr logs with no config code.
- You want the ubiquitous `RUST_LOG` convention that Rust developers already know.
- You need per-test log capture with a one-line init.

**Avoid when:**
- You need structured/contextual logging, spans, or per-request correlation — use `tracing`.
- You need file output, rotation, or multiple sinks — use `log4rs`, `flexi_logger`, or `fern`.
- You log at high volume across many threads and the per-call lock would contend.
- You need to parse log output programmatically — the format is not stable.

## Alternatives

- tokio-rs/tracing — structured, span-aware logging and diagnostics; use instead when you need contextual fields, async instrumentation, or machine-readable output. Its `tracing-subscriber` ships a `RUST_LOG`-compatible `EnvFilter`.
- estk/log4rs — config-file (YAML) driven with file appenders and rotation; use when ops wants to change logging without a rebuild.
- emabee/flexi_logger — `env_logger`-style specs plus file logging and rotation; use when you want familiar ergonomics but need files on disk.
- daboross/fern — builder-style programmatic configuration with multiple outputs; use when you want explicit code-defined dispatch rather than an env var.
- seanmonstar/pretty_env_logger — thin wrapper over `env_logger` with a prettier default format; use for nicer CLI output with the same behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-07-06 | Split from `rust-lang` into its own repo under the CLI working group[^2]. |
| 0.5 | 2018 | Major API reshape around the `Builder` type. |
| 0.10 | 2022 | Moved off the `atty` crate (soundness advisory); MSRV bump. |
| 0.11 | 2024 | `anstyle`/`anstream` migration, `env_filter` extracted, color/timestamp/regex made optional features[^4]. |

## References

[^1]: Repository metadata via GitHub API (`repos/rust-cli/env_logger`), retrieved 2026-07-15: 1,057 stars, 148 forks, license reported `Apache-2.0`, last push 2026-07-09. The published crate is dual-licensed MIT OR Apache-2.0 per its `Cargo.toml` license field. https://github.com/rust-cli/env_logger
[^2]: Repository `created_at` 2017-07-06 (GitHub API), marking the split of `env_logger` out of the `rust-lang` org into the `rust-cli` working group.
[^3]: `env_filter` crate — the `RUST_LOG` directive parser extracted from `env_logger`. https://docs.rs/env_filter
[^4]: `env_logger` CHANGELOG (0.11 dependency overhaul: `anstream`/`anstyle`, `env_filter` split, feature gating). https://github.com/rust-cli/env_logger/blob/main/CHANGELOG.md

## Tags

rust, logging, log-facade, cli, env-var, rust-log, stderr, observability, developer-tools, env-filter
