# SergioBenitez/Figment

> A semi-hierarchical configuration library for Rust that merges layered providers into a single Serde-deserializable value while tracking where every setting came from.

[GitHub repo](https://github.com/SergioBenitez/Figment) ·
[Documentation](https://docs.rs/figment) ·
[License: Apache-2.0 OR MIT](https://github.com/SergioBenitez/Figment#license)

## Overview

Figment is a Rust configuration library written by Sergio Benitez, author of the Rocket web framework[^1]. The problem it solves: an application needs to assemble its final configuration from several sources of differing precedence — a bundled default, a TOML or JSON file, environment variables, CLI-supplied overrides — and then hand a strongly typed struct to the rest of the program. Figment models each source as a `Provider`, layers them with explicit precedence, and deserializes the merged result into any `serde::Deserialize` type.

The defining design choice is **provenance tracking**. Every value carries `Metadata` describing which provider produced it and, where possible, the file and key path. When deserialization fails — a missing field, a string where an integer was expected — the error names the offending source rather than emitting a generic Serde message. This is the feature that distinguishes Figment from thinner "load a file into a struct" crates, and it is why Rocket adopted it as its configuration backend in the 0.5 series[^2].

The tradeoff is conceptual weight. Figment introduces its own vocabulary — providers, profiles, `merge` versus `join`, an internal `Value` enum, `Metadata`, `Jail` — that a caller must learn before the layering behaves predictably. For an app that reads one file into one struct, that surface area is overhead; for an app with defaults plus files plus env plus per-environment profiles, it is the point.

## Getting Started

```toml
[dependencies]
figment = { version = "0.10", features = ["toml", "env"] }
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::Deserialize;
use figment::{Figment, providers::{Format, Toml, Env}};

#[derive(Deserialize)]
struct Config {
    name: String,
    port: u16,
    log_level: Option<String>,
}

fn main() -> Result<(), figment::Error> {
    let config: Config = Figment::new()
        .merge(Toml::file("App.toml"))       // lower precedence
        .merge(Env::prefixed("APP_"))        // env overrides the file
        .extract()?;                         // deserialize into Config

    println!("{} on :{}", config.name, config.port);
    Ok(())
}
```

Later `merge` calls win: `APP_PORT=9000` overrides the `port` set in `App.toml`. Use `join` instead of `merge` when a provider should only supply keys that are not already present.

## Architecture / How It Works

A `Figment` is an ordered stack of providers plus an active profile. Extraction happens in three conceptual stages:

1. **Collection.** Each `Provider` yields a `Map<Profile, Dict>` — its data, bucketed by profile. Built-in providers include `Toml`, `Json`, `Yaml`, and `Env` (each format behind its own Cargo feature), plus `Serialized` for turning an existing struct into a defaults layer.
2. **Combination.** Providers are folded together in insertion order. `merge` gives the incoming provider higher precedence (its values overwrite); `join` gives it lower precedence (it fills only absent keys). The result is a single `Value` tree — Figment's Serde-agnostic intermediate representation — with `Metadata` attached to each node recording its origin.
3. **Extraction.** The combined `Value` for the selected profile is deserialized into the target type via `extract()`. Because the tree is `Value`-typed rather than raw text, coercion (e.g. an env string `"8080"` into a `u16`) happens during this Serde pass.

**Profiles** are Figment's answer to per-environment config. A `Profile` namespaces a set of keys; `Figment::select("production")` picks one at extraction time. Two profiles are special: `Profile::Default` ("default") supplies values when no more specific profile provides them, and `Profile::Global` ("global") applies on top of *every* profile. Getting the interaction of these two right is the single most common source of confusion in Figment code.

**Env parsing** maps variable names to nested keys: `Env::prefixed("APP_")` strips the prefix and lowercases, and `.split("__")` turns `APP_DB__HOST` into `db.host`. Values remain strings until the Serde extraction stage coerces them.

**`Jail`** is a test harness (`figment::Jail::expect_with`) that runs a closure inside a scratch directory with controlled environment variables, then restores both afterward — the supported way to test config assembly without leaking global state between tests.

## Production Notes

- **`merge` vs `join` precedence is the primary footgun.** The mental model "last merge wins" is correct, but mixing `join` and `merge` in one chain makes precedence non-obvious. When a value doesn't take effect, the cause is almost always ordering, not parsing. `Figment::metadata()` and the per-value provenance are the debugging tools — use them before adding print statements.
- **Env coercion can surprise.** Because env values arrive as strings and are coerced during deserialization, whether `APP_FLAG=false` becomes a `bool` or a `String` depends on the target field's type. Collections and maps from env require the documented nesting/splitting syntax; ad-hoc comma strings will not deserialize into a `Vec` without a custom setup.
- **Feature flags gate every format.** `figment` with no features gives you the core plus `Env`; TOML, JSON, and YAML each require enabling their feature. A missing feature is a compile error, not a runtime fallback — expected, but a frequent first-build stumble.
- **YAML support depends on the wider Rust YAML situation.** `serde_yaml`, the ecosystem's dominant YAML backend, is deprecated and unmaintained; teams standardizing on Figment's YAML provider should weigh that upstream risk and consider TOML/JSON for new projects.
- **Maintenance cadence is slow and deliberate.** The 0.10 line has been API-stable for years and the repository sees infrequent commits (last push 2024-09). This reads as "mature and finished," not "abandoned" — but do not expect rapid feature turnaround, and pin to `0.10` rather than tracking a moving target.
- **Still pre-1.0.** The crate has never shipped a 1.0. In practice the 0.10 API is treated as stable and Rocket depends on it, but the version number formally reserves the right to break.

## When to Use / When Not

**Use when:**
- Your configuration comes from more than one source and precedence between them must be explicit.
- You want config errors that name the file and key at fault, not opaque Serde messages.
- You already use Rocket, or want the same configuration model Rocket uses.
- You need per-environment profiles (dev/staging/prod) resolved from one layered definition.

**Avoid when:**
- You read a single file into a single struct — `confy` or a direct `toml`/`serde` call is less to learn.
- Your config is purely environment variables — `envy` deserializes env into a struct with far less surface area.
- Config is driven primarily by CLI flags — reach for `clap`'s derive first and layer Figment only if files/env join in.
- You want a crate under active feature development; Figment is in stable-maintenance mode.

## Alternatives

- mehcode/config-rs — the most common layered-config alternative; similar provider stacking but without Figment's per-value provenance metadata.
- rust-cli/confy — trivial load-and-save of one config file to the platform config dir; use when you just want a struct persisted, not layered.
- softprops/envy — deserialize environment variables straight into a struct; use when env is your only source.
- allan2/dotenvy — loads a `.env` file into the process environment; complementary rather than competing, pair it with env-based config.
- clap-rs/clap — argument parsing with derive; use when CLI flags are the primary configuration surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2020-10 | Initial release; repository created 2020-10-10[^3]. |
| 0.10.x | 2021– | Long-lived current major series: `Provider`/`Profile`/`Metadata` model, `Jail` test harness, TOML/JSON/YAML/Env providers behind features. |

## References

[^1]: Figment README and crate description, `SergioBenitez/Figment`. https://github.com/SergioBenitez/Figment
[^2]: Rocket configuration guide, which documents Figment as Rocket's configuration backend. https://rocket.rs/guide/v0.5/configuration/
[^3]: GitHub repository metadata (`created_at` 2020-10-10; last push 2024-09-13). https://github.com/SergioBenitez/Figment

## Tags

rust, configuration, config-management, serde, environment-variables, toml, layered-config, library, rocket, cli
