# rust-lang/cargo

> Rust's official package manager and build orchestrator — manifest, resolver, and build system in one binary that ships with the toolchain.

[GitHub repo](https://github.com/rust-lang/cargo) ·
[Official website](https://doc.rust-lang.org/cargo) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/cargo/blob/master/LICENSE-APACHE)

## Overview

Cargo is the build tool and package manager that ships with every Rust toolchain. It resolves dependencies from a registry (crates.io by default), fetches and compiles them, drives `rustc`, and exposes a uniform command surface (`build`, `test`, `run`, `check`, `publish`) across the entire ecosystem[^1]. Unlike languages where build tooling is a fragmented afterthought (Python, C++, JavaScript), Rust shipped Cargo alongside 1.0 in 2015, so effectively 100% of the ecosystem uses the same manifest format, the same lockfile, and the same registry conventions. That uniformity is Cargo's defining property and the reason Rust tooling feels coherent.

Cargo is developed in-tree with the rest of the Rust project and released in lockstep with the compiler on a six-week cadence — Cargo versions track Rust versions (Rust 1.85 ships Cargo 1.85). It is not independently versioned or independently installable; you get the Cargo that matches your `rustc`. As of 2026 the repository has ~15.2k stars and ~2.9k forks, with an open-issue count near 1,650 that reflects its scope as much as its bug load: Cargo is simultaneously a resolver, a build scheduler, a registry client, a workspace manager, and the coordination point for build scripts and features[^1].

The repo's own README carries an unusually blunt disclaimer: the `cargo` crate is maintained primarily for use *by Cargo*, not as a general-purpose library, and may make major API changes at will[^2]. Consume Cargo through its CLI, not by linking the crate.

## Getting Started

Cargo installs with the toolchain via rustup; you rarely install it separately.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

```bash
cargo new my-app          # scaffold a binary crate (--lib for a library)
cd my-app
cargo add serde --features derive   # edit Cargo.toml + resolve
cargo run                 # build (dev profile) and execute
cargo test                # build + run unit/integration/doc tests
cargo build --release     # optimized build into target/release/
```

A minimal `Cargo.toml`:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"     # MSRV — enforced by the resolver

[dependencies]
serde = { version = "1", features = ["derive"] }

[profile.release]
lto = true
codegen-units = 1
```

## Architecture / How It Works

**Manifest + lockfile.** `Cargo.toml` is the human-authored manifest (SemVer version requirements, features, profiles). `Cargo.lock` is the machine-generated resolution — exact versions and checksums. The long-standing guidance to commit `Cargo.lock` for binaries but not libraries has been superseded: committing it for both is now recommended for reproducible CI[^3].

**Resolution.** Cargo builds a dependency graph and picks the newest SemVer-compatible version of each crate, allowing multiple *incompatible* major versions to coexist in one build. The resolver has three generations: v1 (legacy), v2 (default for edition 2021, changed how features unify across build/dev/target dependencies), and v3 — the MSRV-aware resolver stabilized in Rust 1.84 (2025), which prefers dependency versions compatible with your declared `rust-version`[^4].

**Features.** Conditional compilation flags declared per-crate. Their most-cited footgun is *feature unification*: within a build, Cargo takes the union of all features requested for a given crate by anyone in the graph. A dependency you pull in for one binary can silently enable a feature in a shared crate, changing what another target compiles. Resolver v2 narrowed but did not eliminate this[^4].

**Build scripts.** `build.rs` runs before compilation for codegen, native-library discovery, and `cfg` emission. Build scripts are arbitrary Rust that runs on the build machine — a real supply-chain surface, since `cargo build` alone executes third-party code.

**Registry client.** Since Rust 1.70 (2023) the default crates.io protocol is the *sparse* HTTP index, which fetches only the index entries a build needs instead of cloning the full git index — a large improvement in first-build and update latency[^5]. Downloads and the compiled git/HTTP machinery historically leaned on vendored `libcurl`, `libgit2`, and `libssh2` C libraries.

**Workspaces.** Multiple crates share one `target/` directory, one `Cargo.lock`, and (since Rust 1.64) inheritable dependency and metadata tables via `[workspace.dependencies]`[^6].

## Production Notes

**Disk usage is the quiet operational cost.** `target/` is per-project and unbounded; on a machine with many Rust checkouts it routinely reaches tens of gigabytes. There is no built-in global build cache — `cargo clean`, `cargo-sweep`, or an external cache (`sccache`) is the mitigation. The download cache under `~/.cargo/registry` also grows without bound.

**Build times dominate the Rust experience, and Cargo is the scheduler.** Use `cargo check` (skips codegen) during iteration; reserve `cargo build` for when you need a binary. Compilation is parallel across crates but a single large crate is a serialization bottleneck — splitting a workspace into smaller crates improves incremental rebuild parallelism. `codegen-units` and `lto` in the release profile trade compile time for runtime speed.

**crates.io is append-only.** You cannot delete a published version; `cargo yank` only prevents *new* dependency resolutions from selecting it — existing `Cargo.lock` files still resolve. Publish carefully; a leaked secret in a published crate is public permanently.

**`cargo install` is not a system package manager.** It compiles binaries from source into `~/.cargo/bin` and does not track or upgrade them as a set (`cargo-install-update` fills that gap). Installs pin to the crate's own lockfile only with `--locked`; by default they re-resolve, which can pull newer transitive versions than the author tested.

**Feature-unification surprises in CI.** A workspace that builds fine locally can fail or bloat when a dev-dependency's features leak into a release target. Auditing with `cargo tree -e features` before shipping is the standard defense.

**MSRV drift.** Before the v3 resolver, a routine `cargo update` could pull a transitive dependency that raised your effective minimum Rust version and break consumers on older toolchains. Setting `rust-version` and using resolver v3 (Rust 1.84+) addresses this; older toolchains do not benefit[^4].

## When to Use / When Not

**Use when:**
- You are writing any Rust — Cargo is the ecosystem default and opting out fights the entire tooling stack.
- You want reproducible builds via a committed lockfile and vendored dependencies (`cargo vendor`).
- You need workspaces to manage a multi-crate project with a shared lockfile and target dir.

**Reach for a supplement when:**
- Your test suite is large or flaky under shared process state — `cargo test` runs tests in-process by thread; nextest gives per-test process isolation and faster scheduling.
- You need real build pipelines, task orchestration, or cross-tool workflows — Cargo aliases are thin; a task runner does more.
- You need a global compilation cache across projects or CI — Cargo has none natively; add `sccache`.

## Alternatives

There is effectively no drop-in replacement for Cargo *within* Rust; the honest "alternatives" are complements and cross-ecosystem peers.

- nextest-rs/nextest — use instead of `cargo test` when you want process-isolated, faster, better-reported test runs.
- sagiegurari/cargo-make — use when you need a genuine task runner and multi-step build pipeline beyond Cargo aliases.
- casey/just — use for project command scripting that isn't a build (formatting, deploys, chores).
- astral-sh/uv — not for Rust; the closest thing to Cargo's UX brought to Python packaging.
- golang/go — the integrated build-plus-dependency toolchain that plays Cargo's role in Go.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-03 | Development begins in-tree; initial design by Yehuda Katz and Carl Lerche. |
| 1.0 | 2015-05 | Ships with Rust 1.0; stable manifest + lockfile[^1]. |
| 1.12 | 2016-09 | Workspaces stabilized. |
| 1.31 | 2018-12 | Edition 2018; `cargo fix` for edition migration. |
| 1.51 | 2021-03 | Resolver v2 available (opt-in via `resolver = "2"`)[^4]. |
| 1.56 | 2021-10 | Edition 2021 makes resolver v2 the default; `rust-version` (MSRV) field. |
| 1.62 | 2022-06 | `cargo add` merged into Cargo itself. |
| 1.64 | 2022-09 | Workspace inheritance (`[workspace.dependencies]`)[^6]. |
| 1.70 | 2023-06 | Sparse registry protocol becomes the default for crates.io[^5]. |
| 1.84 | 2025-01 | MSRV-aware resolver v3 stabilized[^4]. |

## References

[^1]: The Cargo Book — official documentation. https://doc.rust-lang.org/cargo/
[^2]: rust-lang/cargo README — maintenance and API-stability disclaimer. https://github.com/rust-lang/cargo/blob/master/README.md
[^3]: The Cargo Book, "Why have Cargo.lock in version control?" https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html
[^4]: The Cargo Book, "Dependency Resolution" (resolver versions, MSRV-aware resolver). https://doc.rust-lang.org/cargo/reference/resolver.html
[^5]: Rust Blog, "A new default registry protocol" — sparse index. https://blog.rust-lang.org/2023/03/09/Rust-1.68.0.html
[^6]: The Cargo Book, "Workspaces" — inheritance. https://doc.rust-lang.org/cargo/reference/workspaces.html

## Tags

rust, package-manager, build-system, dependency-resolution, cli, crates-io, toolchain, cargo, systems-programming, developer-tools
