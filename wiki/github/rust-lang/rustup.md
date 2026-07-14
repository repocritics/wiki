# rust-lang/rustup

> The official Rust toolchain installer — manages multiple `rustc`/`cargo` versions, channels, targets, and components behind a set of PATH shims.

[GitHub repo](https://github.com/rust-lang/rustup) ·
[Official website](https://rustup.rs/) ·
[License: Apache-2.0 OR MIT](https://github.com/rust-lang/rustup/blob/master/LICENSE-APACHE)

## Overview

Rustup is the program that most Rust developers actually run first: the one-line `curl … | sh` from rustup.rs installs it, and it in turn installs the Rust compiler and Cargo[^1]. It is a toolchain *multiplexer* — it holds any number of installed toolchains (stable, beta, nightly, or pinned versions like `1.85.0`), lets you switch the default, and layers on cross-compilation targets and optional components (clippy, rustfmt, rust-analyzer, rust-src, rust-docs). It runs on every platform Rust supports, including Windows with both the MSVC and GNU ABIs.

The design is deliberately invisible in daily use. After install, `rustc`, `cargo`, `clippy`, and friends on your PATH are not the real binaries — they are thin shim executables in `~/.cargo/bin` that decide *which* toolchain to dispatch to based on the current directory, an override, or the `+toolchain` argument. This indirection is what makes `cargo +nightly build` and per-project pinning work without any environment juggling. It is also the source of most of rustup's surprising behavior, because the tool you think you are running is one hop away from the tool that actually runs.

Rustup is maintained under the rust-lang organization and is the officially recommended installation path[^1], but it is not the compiler and not part of `rustc`'s release train — it versions independently and updates itself (`rustup self update`). Its predecessor was `multirust`; rustup replaced it as the canonical installer in 2016.

## Getting Started

```bash
# Unix / macOS
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Windows: download and run rustup-init.exe from https://rustup.rs
```

Common operations:

```bash
rustup update                       # update all installed toolchains
rustup toolchain install nightly    # add the nightly channel
rustup default stable               # set the global default
rustup component add clippy rustfmt # add optional components
rustup target add wasm32-unknown-unknown   # add a cross-compile target
cargo +nightly build                # run one command under a specific toolchain
```

Pin a toolchain per project with a `rust-toolchain.toml` at the repo root — rustup reads it automatically and installs the toolchain on first use[^2]:

```toml
[toolchain]
channel = "1.85.0"
components = ["clippy", "rustfmt"]
targets = ["wasm32-unknown-unknown"]
```

## Architecture / How It Works

Rustup has three moving parts:

1. **The shims.** `~/.cargo/bin/` contains proxy binaries (`rustc`, `cargo`, `rustfmt`, `cargo-clippy`, …) that are all the same rustup executable under different names. When invoked, a shim resolves the *active toolchain* and re-execs the real binary inside `~/.rustup/toolchains/<name>/bin/`.
2. **Toolchain resolution.** Precedence, highest first: an explicit `+toolchain` argument; the `RUSTUP_TOOLCHAIN` env var; a directory override set with `rustup override set`; a `rust-toolchain.toml` (or legacy `rust-toolchain`) file found by walking up from the CWD; otherwise the global default.
3. **The dist protocol.** Toolchains are downloaded from static release servers (`static.rust-lang.org` by default, overridable via `RUSTUP_DIST_SERVER`). Each channel publishes a manifest listing available components and per-target artifacts; rustup fetches the manifest, resolves the requested profile, downloads the `.tar.xz` component archives, and verifies SHA-256 checksums against the manifest.

**Profiles** control how much gets installed: `minimal` (rustc + cargo + std, ideal for CI), `default` (adds rustfmt, clippy, docs), and `complete` (everything, not recommended — it will frequently fail on nightly because some component didn't build that day).

**Components vs targets** are distinct axes. A *component* (clippy, rust-src) is a tool or data set for the host. A *target* (`rustup target add …`) is a precompiled copy of `std` for a cross-compilation platform. Both are keyed to a specific toolchain, so adding nightly does not carry over the targets you added to stable.

State lives in two directories: `~/.rustup` (installed toolchains, downloads, settings) and `~/.cargo/bin` (the shims plus anything `cargo install` puts there). `RUSTUP_HOME` and `CARGO_HOME` relocate them.

## Production Notes

**The shim indirection leaks.** `which rustc` reports `~/.cargo/bin/rustc`, not the real compiler; tools that introspect the binary path, sandboxes that whitelist executables, or debuggers can be confused. `rustup which rustc` prints the resolved real path.

**`rust-toolchain.toml` triggers silent downloads.** Cloning a repo and running any cargo command can cause rustup to download an entire pinned toolchain the first time — surprising on metered connections, in CI without a cache, or in air-gapped environments. In CI, pin deliberately and cache `~/.rustup` and `~/.cargo`.

**Nightly component roulette.** On any given nightly, a component (clippy, rust-docs, miri, rust-analyzer) may be missing because it failed to build. `rustup toolchain install nightly --component clippy` then errors. `rustup update` will hold a component back rather than break the toolchain. Teams that need a nightly with a specific component sometimes pin to a dated nightly (`nightly-2026-06-01`) known to have it.

**Disk growth.** Each toolchain is hundreds of MB and each added target adds more; nightlies accumulate. There is no aggressive automatic garbage collection of old toolchains — prune with `rustup toolchain uninstall` (recent versions add a `rustup gc`-style cleanup, but old installs still bloat `~/.rustup`).

**Corporate proxies and MITM TLS.** The most common install failure in enterprises. A proxy that re-signs TLS breaks certificate validation; `RUSTUP_DIST_SERVER` / `RUSTUP_UPDATE_ROOT` point at internal mirrors, and `RUSTUP_USE_CURL` / native-TLS vs rustls builds sometimes matter for custom CA stores.

**Signature verification.** Rustup verifies SHA-256 checksums from the manifest and relies on HTTPS transport integrity for the manifest itself; it does not perform end-to-end GPG signature verification of toolchain artifacts by default. For high-assurance supply chains this is worth understanding rather than assuming.

**Windows specifics.** Rustup must choose the MSVC (needs Visual Studio Build Tools) or GNU ABI; the wrong choice produces link errors. PATH precedence between `~/.cargo/bin` and system-installed Rust packages is a recurring footgun (a 2022 advisory concerned rustup's binary lookup path).

## When to Use / When Not

**Use when:**
- You develop Rust and want to switch channels, pin per-project toolchains, or cross-compile.
- You need reproducible toolchains in CI (`rustup toolchain install --profile minimal`).
- You want the officially supported, cross-platform install path including Windows.

**Avoid / reconsider when:**
- You want fully reproducible, hermetic builds — a Nix-based overlay pins toolchains by hash more strictly than a rustup channel.
- You only ever need one system-wide Rust and your distro package is recent enough — `apt`/`dnf`/`brew` avoid the shim layer.
- You are in a locked-down air-gapped environment where an internal mirror plus vendored toolchains is simpler than rustup's dist protocol.

## Alternatives

- rust-lang/rust — the compiler and std that rustup actually installs; you can build/install it directly without rustup, losing channel management.
- oxalica/rust-overlay — Nix overlay for hash-pinned, reproducible Rust toolchains; the hermetic-build alternative.
- nix-community/fenix — another Nixpkgs-based Rust toolchain provider with rust-analyzer nightly support.
- jdx/mise (and asdf) — polyglot version managers with a Rust plugin; use when you already standardize language versions across a team and want one tool.
- Distribution packages (apt/dnf/pacman/homebrew) — use when you need exactly one, distro-blessed Rust and don't want the shim indirection.

## History

| Version | Date | Notes |
|---------|------|-------|
| multirust | ~2014 | Predecessor shell/tool by Brian Anderson; managed multiple toolchains. |
| rustup 1.0.0 | 2016 | Rust-native rewrite; became the official recommended installer, replacing multirust. |
| 1.23.0 | 2020-12 | `rust-toolchain.toml` structured override format[^2]. |
| 1.25.0 | 2022 | Security-relevant fix around binary lookup path precedence. |
| 1.26.0 | 2023 | Continued maintenance; profile/component handling improvements. |
| 1.27.x | 2024 | Ongoing maintenance releases. |

## References

[^1]: The Rust Programming Language, "Installation" — recommends rustup. https://doc.rust-lang.org/book/ch01-01-installation.html
[^2]: The Rustup Book, "Overrides" and the `rust-toolchain.toml` format. https://rust-lang.github.io/rustup/overrides.html

## Tags

rust, toolchain, version-manager, installer, cli, cross-compilation, developer-tools, cargo, package-management, systems
