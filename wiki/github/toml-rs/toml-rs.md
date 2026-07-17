# toml-rs/toml-rs

> The original home of Rust's `toml` crate — now archived, with development moved to the toml-rs/toml workspace.

[GitHub repo](https://github.com/toml-rs/toml-rs) ·
[Docs](https://docs.rs/toml) ·
[License: Apache-2.0](https://github.com/toml-rs/toml-rs)

## Overview

`toml-rs` was the original repository for the `toml` crate, the most widely used
TOML encoding/decoding library in the Rust ecosystem[^1]. It was created in 2014
and for most of its life housed a single crate that parsed TOML into a `Value`
enum or, via `serde`, directly into user structs. If you have ever written
`toml::from_str` in Rust, this is where that code originated.

This repository is **archived**. As its README states, development of the `toml`
crate moved to the multi-crate workspace at `toml-rs/toml`[^2]. The last commit
here landed in September 2022, and the crate's own major rewrite (0.6, on top of
`toml_edit`) shipped from the successor repository, not this one. The stars and
forks on this page reflect a decade of accumulated history, not current
activity — treat any issue tracker or code here as frozen.

The practical takeaway: `toml-rs/toml-rs` is of historical interest only. For a
current dependency, the crate name is still `toml` on crates.io, but the source,
issues, and releases live in `toml-rs/toml`. This page documents the crate the
repository produced, and points you to where it went.

## Getting Started

The crate is still published as `toml`; only the repository moved. Add it
alongside `serde`:

```bash
cargo add toml
cargo add serde --features derive
```

Deserialize a TOML document into a typed struct:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Config {
    title: String,
    owner: Owner,
}

#[derive(Deserialize)]
struct Owner {
    name: String,
}

fn main() {
    let src = r#"
        title = "example"

        [owner]
        name = "Tom"
    "#;

    let cfg: Config = toml::from_str(src).unwrap();
    println!("{}", cfg.owner.name); // Tom
}
```

Serialization is symmetric: `toml::to_string(&value)` produces a TOML document
from any `Serialize` type. For untyped access, `toml::Value` is an enum over the
TOML types (string, integer, float, boolean, datetime, array, table).

## Architecture / How It Works

The `toml` crate sits on top of `serde`'s data model. Parsing goes text →
internal document → either a `toml::Value` tree or, through the `serde`
`Deserializer`, straight into user types. Serialization runs the reverse path.
TOML's value types (notably its first-class datetimes) do not map cleanly onto
JSON-shaped serde, so the crate ships its own `toml_datetime` types rather than
reusing strings.

The significant architectural fact is the **0.6 rewrite**: from that version
on, `toml` is implemented on top of `toml_edit`[^3], a format- and
style-preserving TOML parser that keeps whitespace, comments, and ordering. The
older 0.5 line used a bespoke parser that discarded formatting. This split is
why the modern project is a workspace rather than a single crate:

- `toml` — the serde-facing encode/decode crate most users depend on.
- `toml_edit` — the format-preserving document model; used by Cargo itself for
  `cargo add` and other edits to `Cargo.toml`[^3].
- `toml_datetime` — the RFC 3339-style datetime types shared across the above.
- `serde_spanned` — carries source span information through serde so callers can
  report error locations back to the original file.

That reuse of `toml_edit` under `toml` is the coupling story: a single parser now
backs both "give me my struct" and "edit this file without touching its
formatting," which reduced divergence between the two but also raised the
crate's minimum-supported-Rust-version floor and its dependency surface compared
to the lean 0.5 era.

## Production Notes

- **Pin to the successor, not this repo.** Depending on `toml` from crates.io
  already pulls the maintained code; there is nothing to do here except not file
  issues against the archived tracker. Bug reports and feature requests belong in
  `toml-rs/toml`.
- **0.5 → 0.6 is a semver break with behavior changes.** The move onto
  `toml_edit` changed some error messages, span reporting, and edge-case
  formatting of emitted documents. Upgrading across that boundary warrants
  re-checking any snapshot tests that assert on serialized output.
- **Datetimes are not chrono.** `toml` exposes its own `Datetime` type via
  `toml_datetime`. If your structs use `chrono` or `time`, you deserialize into
  the toml datetime and convert, or use a serde adapter — there is no built-in
  bridge.
- **Serialization ordering.** When serializing a struct, field order follows the
  struct definition; when round-tripping through `toml::Value` (a map), key
  ordering depends on the map type. For deterministic output, serialize from a
  typed struct rather than a `Value`.
- **Heavier than it looks for simple needs.** For a program that only needs to
  read a small fixed config once, the full serde + `toml_edit` stack is more
  dependency weight than some projects want; see alternatives below.

## When to Use / When Not

**Use `toml` (the crate) when:**
- You want serde-based config or manifest parsing that is the ecosystem default.
- You need spec-compliant TOML with correct datetime and nested-table handling.
- You will also edit TOML files in place and want the `toml_edit` sibling.

**Avoid / look elsewhere when:**
- You are reading this repository expecting current code — it is archived; go to
  `toml-rs/toml`.
- You want a minimal-dependency deserializer for read-only config — `basic-toml`
  is smaller.
- You need TOML tooling (formatter, linter, language server), not a library —
  reach for the `taplo` toolkit.

## Alternatives

- toml-rs/toml — the active successor workspace; use this instead, always, for
  any current dependency or issue.
- tamasfe/taplo — full TOML toolkit (formatter, validator, language server) when
  you need tooling rather than an embed-in-your-program library.
- dtolnay/basic-toml — a minimal fork of the 0.5-era parser; use it when you want
  read/write TOML with a smaller dependency tree and no format-preserving editing.
- mehcode/config-rs — layered configuration across TOML, env, and other sources
  when a single TOML file is not enough.
- SergioBenitez/Figment — figment-based layered config when you want profiles and
  merging on top of format parsers.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2014-06 | Original `toml-rs` repository opened. |
| 0.5.x | 2019 | Long-lived line; bespoke parser, no format preservation. |
| repo archived | 2022-09 | Development moved to the `toml-rs/toml` workspace[^2]. |
| 0.6 | 2023 | Crate rewritten on top of `toml_edit`; shipped from successor repo[^3]. |

Versions from 0.6 onward were released from `toml-rs/toml`; the entries above are
listed here only to explain what happened to the code this repository once held.

## References

[^1]: `toml` crate on crates.io. https://crates.io/crates/toml
[^2]: toml-rs/toml-rs README — "Development of the `toml` crate has moved to
      https://github.com/toml-rs/toml/tree/master/crates/toml. This repo is now
      archived." https://github.com/toml-rs/toml-rs
[^3]: `toml_edit` — format-preserving TOML for Rust, used by Cargo. https://docs.rs/toml_edit

## Tags

rust, toml, serde, parser, serialization, config, encoding, archived, crates-io
