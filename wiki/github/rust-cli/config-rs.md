# rust-cli/config-rs

> Layered configuration for Rust: merge defaults, files in several formats, and environment variables into one serde-deserializable tree.

[GitHub repo](https://github.com/rust-cli/config-rs) ·
[Documentation](https://docs.rs/config) ·
[License: Apache-2.0 OR MIT](https://github.com/rust-cli/config-rs/blob/main/LICENSE-APACHE)

## Overview

`config-rs` (published as the `config` crate) assembles application configuration from ordered layers — hard-coded defaults, one or more config files, environment variables, and explicit programmatic overrides — and merges them into a single value tree that you read directly or deserialize into your own struct via serde. It targets [12-factor](https://12factor.net/config)-style applications where the same binary is configured differently per environment, with env vars taking precedence over checked-in files.[^1]

The crate was originally written by Ryan Leckey (`@mehcode`) and is now maintained under the `rust-cli` GitHub organization; the old `mehcode/config-rs` path redirects there. It is one of the older and more widely depended-on configuration crates in the Rust ecosystem, pulled in transitively by a large number of CLI tools and services.

The defining tension is *loose typing over static guarantees*. Values are stored in an internal dynamic `Value` type and coerced on read ("any supported type, as long as a reasonable conversion exists"), which is what makes format-agnostic layering possible — but it also pushes a class of type-mismatch errors from compile time to run time, surfacing only when `try_deserialize` runs. A second deliberate limitation: the library reads configuration only. It cannot write changed values back to source files.[^2]

## Getting Started

```toml
# Cargo.toml — toml support is on by default; other formats are feature-gated
[dependencies]
config = "0.14"
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::Deserialize;
use config::{Config, Environment, File};

#[derive(Debug, Deserialize)]
struct Settings {
    debug: bool,
    database: Database,
}

#[derive(Debug, Deserialize)]
struct Database {
    url: String,
    pool_size: u32,
}

fn main() -> Result<(), config::ConfigError> {
    let settings = Config::builder()
        // 1. base file (config.toml)
        .add_source(File::with_name("config"))
        // 2. optional per-env file
        .add_source(File::with_name("config.local").required(false))
        // 3. env vars: APP_DATABASE__URL -> database.url
        .add_source(Environment::with_prefix("APP").separator("__"))
        .set_override("debug", true)?   // highest precedence
        .build()?;

    let settings: Settings = settings.try_deserialize()?;
    println!("{settings:?}");
    Ok(())
}
```

Later sources win over earlier ones. `set_override` beats every added source; `set_default` is the floor beneath them.

## Architecture / How It Works

The core abstraction is the `Source` trait: each source (a file, an env-var set, an in-memory map) knows how to produce a flat `Map<String, Value>`. A `ConfigBuilder` holds an ordered list of sources plus explicit defaults and overrides. Calling `.build()` collects every source in order and deep-merges them into a single `Config`, resolving precedence as it goes.[^3]

`Value` is a dynamic enum (string, int, float, bool, array, table, nil). Deserialization into your struct is a serde pass over that tree, so any `Deserialize` type works, and unknown/extra keys are ignored unless you opt into `deny_unknown_fields`. Reading individual keys uses a **path expression** — a subset of JSONPath supporting the child operator (`redis.port`) and array subscript (`databases[0].name`).[^1]

File formats are pluggable through the `Format` trait and gated behind Cargo features: `toml`, `json`, `yaml`, `ini`, `ron`, `json5`, and `corn`. Each maps its parsed document into the common `Value` tree, which is why a YAML file and a TOML file can layer on top of each other transparently. Custom/proprietary formats can be added by implementing `Format` yourself.

Two structural facts drive most real-world behavior:

- **Merge is deep for tables, replace for arrays.** Overlapping maps merge key-by-key across layers; arrays are replaced wholesale by the higher-precedence layer, not concatenated or index-merged. This routinely surprises people expecting to append to a list from an env override.
- **The env source flattens.** Environment variables are strings with no nesting, so nested keys are reconstructed by splitting on a configured `separator` (commonly `__`). Everything arrives as a string and is coerced on demand, so a numeric env var is a `String` in the tree until something reads it as an integer.

The older mutable `Config::new()` + `merge()` API was replaced by the immutable `ConfigBuilder` pattern (`Config::builder()...build()`) in the 0.11 line; the old style was deprecated and later removed. Code and tutorials written against pre-0.11 will not compile on current versions.[^4]

## Production Notes

- **The pre-0.11 → builder migration is the single biggest upgrade cost.** Any codebase or blog post using `Config::new()`, `merge()`, or `Config::default()` mutation must be rewritten to the builder API. Budget for it; it is not a mechanical find-replace because precedence ordering is now expressed by add-order rather than merge-call order.
- **Env var casing and separators are footgun-dense.** By default keys are lowercased; the nesting `separator` and the prefix `separator` are distinct concerns, and getting `APP_DATABASE__URL` to land at `database.url` requires matching `with_prefix` + `separator` exactly. Parsing lists from env (`try_parsing(true)` + `list_separator`) is off by default.
- **Loose typing defers errors to runtime.** A YAML `port: "5432"` (quoted) versus `port: 5432` can both "work" until a consumer demands a specific type. Prefer a single `try_deserialize` into a typed struct at startup so mismatches fail fast and in one place, rather than scattered `get::<T>` calls.
- **No write-back, by design.** If you need to persist edited settings, this is the wrong layer — pair it with your own serializer or use a crate built around round-tripping.
- **File discovery is explicit.** `File::with_name("config")` searches for any supported extension in the working directory; it does not walk up parent directories or consult XDG paths. There is no built-in "find my config in the usual OS locations" behavior — that is what `confy` and `figment` add on top.
- **Async sources exist but are niche.** An `AsyncSource` trait allows fetching config from remote stores; most deployments use the synchronous file/env path.
- **Format features are additive and can bloat builds.** Enabling `yaml`/`ron`/`json5` pulls their parser crates; keep only the formats you actually parse. `toml` being a default feature means it ships unless you set `default-features = false`.

## When to Use / When Not

**Use when:**
- You want one binary configured across environments with env vars overriding files (12-factor).
- Your config spans multiple formats or you want to switch formats without touching read sites.
- You want to deserialize the merged result into typed structs via serde.
- You need layered precedence (defaults < files < env < CLI overrides) with clear ordering.

**Avoid when:**
- You have exactly one file in one format and no layering — parse it directly with `serde` + `toml`/`serde_json` and skip the abstraction.
- You need to write configuration back to disk or preserve comments/formatting on round-trip.
- You want automatic OS-standard config paths and zero-boilerplate load/store — `confy` is a better fit.
- You want strong-typed, compile-time-oriented layering with profiles — `figment` maps more directly onto that model.

## Alternatives

- SergioBenitez/Figment — layered/profile-based config from the Rocket ecosystem; use instead when you want serde-first extraction with named profiles and richer error provenance.
- rust-cli/confy — minimal load/store of a single serde struct at the OS-standard config path; use instead when you just want "load my settings struct, create it if missing."
- bnjjj/twelf — derive-macro layered config (env, files, clap); use instead when you want the layer wiring generated from a struct attribute rather than built by hand.
- clap-rs/clap — CLI argument parsing; use instead (or alongside) when configuration comes primarily from flags and you want typed derive + help generation, feeding overrides into a layered store.
- serde-rs/serde + a format crate (toml / serde_json / serde_yaml) — use directly when there is a single source and no need to merge layers at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2017 | Initial release by Ryan Leckey (`@mehcode`); mutable `Config` + `merge()` API.[^4] |
| 0.11.0 | 2021 | Immutable `ConfigBuilder` API introduced; old mutable API deprecated.[^4] |
| 0.13.x | 2022 | Ongoing format/serde maintenance; async source support. |
| 0.14.x | 2024 | Format backend updates (YAML moved to `yaml-rust2`), added format features (e.g. `corn`). |

Version cadence is slow and stable; the crate has stayed pre-1.0 while being heavily depended upon. Consult docs.rs and the CHANGELOG for exact per-release details.[^2]

## References

[^1]: config-rs README — feature list, path expressions, format support. https://github.com/rust-cli/config-rs/blob/main/README.md
[^2]: `config` crate documentation. https://docs.rs/config
[^3]: `ConfigBuilder` API reference. https://docs.rs/config/latest/config/builder/struct.ConfigBuilder.html
[^4]: config-rs CHANGELOG (builder API migration, release history). https://github.com/rust-cli/config-rs/blob/main/CHANGELOG.md

## Tags

rust, configuration, config-management, 12-factor, serde, environment-variables, toml, yaml, layered-config, cli
