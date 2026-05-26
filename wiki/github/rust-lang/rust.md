# rust-lang/rust

> The Rust programming language — empowering everyone to build reliable and efficient software.

[GitHub repo](https://github.com/rust-lang/rust) ·
[Official website](https://www.rust-lang.org) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/rust/blob/master/COPYRIGHT)

## Overview

Rust is a systems programming language with an ownership-based memory model, no garbage collector, zero-cost abstractions, and a strong async story[^1]. Originally a Mozilla research project (Graydon Hoare, 2010), Rust 1.0 shipped in 2015. Governance moved to the Rust Foundation in 2021 after Mozilla's reorganization.

The defining feature is **the borrow checker**: at compile time, the compiler verifies that every value has a single owner, references do not outlive the values they point to, and mutable aliasing is forbidden. This eliminates entire classes of memory bugs (use-after-free, double-free, data races on shared memory) without a runtime garbage collector. It also produces a steep learning curve: the compiler will reject programs that would be correct in C++ because it cannot prove safety locally.

Rust is most used in: systems-adjacent tooling (Cargo, rustc itself, ripgrep, fd, bat, eza, ruff, uv, biome, swc, rolldown, oxc), networked services where memory safety + performance both matter (Discord, Cloudflare, Fastly), embedded firmware (the embedded-hal ecosystem), WebAssembly targets, and increasingly OS kernels (Linux kernel Rust support stable since 6.1).

The **edition system** (2015, 2018, 2021, 2024) lets the language make breaking syntactic changes without breaking older crates — each crate declares its edition, and crates of different editions interoperate. Edition 2024 shipped with Rust 1.85.

## Getting Started

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Initialize a project:

```bash
cargo new my-app
cd my-app
cargo run
```

A minimal Rust program with ownership and pattern matching:

```rust
fn main() {
    let users: Vec<User> = vec![
        User { id: 1, name: "Tom".into() },
        User { id: 2, name: "Brad".into() },
    ];

    for user in &users {
        println!("{}", greet(user));
    }
}

struct User {
    id: u32,
    name: String,
}

fn greet(user: &User) -> String {
    format!("Hello, {}", user.name)
}
```

Async with Tokio:

```bash
cargo add tokio --features full
cargo add reqwest --features json
```

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let body = reqwest::get("https://api.example.com").await?.text().await?;
    println!("{body}");
    Ok(())
}
```

## Architecture / How It Works

`rustc` is a multi-stage compiler[^2]:

1. **Parse** — source → AST.
2. **Lowering** — AST → HIR (High-level IR) → THIR → MIR (Mid-level IR).
3. **Borrow check** — runs on MIR; the canonical safety pass. The next-generation "Polonius" checker is in development and accepts strictly more programs.
4. **Optimization** — MIR → LLVM IR → machine code via LLVM backend (with cranelift as an experimental alternative for debug builds).

**Ownership rules** (the borrow checker enforces):
- Each value has exactly one owner.
- Borrowing: any number of immutable references (`&T`) **or** exactly one mutable reference (`&mut T`).
- References must not outlive the data they reference (the lifetime system tracks this).

**Cargo** is the package manager and build orchestrator. `crates.io` is the registry. Workspaces let multi-crate projects share a target directory and lockfile.

**Async/await** is built around the `Future` trait. There is no built-in runtime; `tokio`, `async-std`, and `smol` are the runtime choices. `tokio` is the dominant production choice. Async trait methods became stable in 1.75 (2023) but still have some restrictions around dyn-compatibility.

**`unsafe` Rust** is a strict subset that allows raw pointer dereferencing, calling FFI, mutable statics, and implementing `unsafe` traits. Code in an `unsafe` block must uphold the language's invariants manually; the burden of proof shifts from compiler to author. Most production Rust has very little `unsafe`, concentrated at FFI boundaries.

## Production Notes

**Compile times.** The single most-complained-about aspect of production Rust. Cold builds of medium projects (50k–100k LOC) take 1–3 minutes; large projects (rustc, servo, cargo) take 10+ minutes. Mitigations:
- `cargo check` instead of `cargo build` during development (skips codegen).
- `sccache` or `cachepot` for shared build cache.
- Workspace splitting — smaller compilation units rebuild faster.
- `mold` or `lld` linker on Linux for faster link.
- `cargo-nextest` for faster test execution.
- The cranelift backend (`rustup default nightly` + `cargo +nightly build`) speeds debug builds by ~2×.

**MSRV (Minimum Supported Rust Version).** Library authors declare an MSRV — the oldest Rust version they support. Strategies range from "track stable" to "MSRV is N=12 months behind stable". The `rust-version` field in `Cargo.toml` enforces it. Tools like `cargo-msrv` automate verification.

**Async footguns.**
- `Pin` is the source of most async confusion. Self-referential futures need `Pin` to prevent moves; the `pin-project` macro handles most cases.
- Cancellation safety: dropping a future at any await point may leave state inconsistent. `tokio::select!` requires cancel-safe futures; the standard library docs annotate which `tokio` primitives are cancel-safe.
- `async fn` in traits is stable but `dyn Trait` for async methods still requires `async-trait` or boxing.

**Error handling.** `Result<T, E>` + the `?` operator is the standard. Libraries: `thiserror` for typed errors, `anyhow` for application-level error context. `Box<dyn Error>` is acceptable for binaries.

**FFI / `unsafe`.** Required for C interop, hardware access, and some performance-critical code. Review patterns: minimize `unsafe` block scope, document safety invariants with `// SAFETY:` comments, use `miri` to catch undefined behavior in tests.

**Embedded.** The `no_std` ecosystem is mature for many MCU families (Cortex-M, RISC-V, ESP32 via `esp-rs`). HAL crates and `embassy` (async embedded executor) dominate.

## When to Use / When Not

**Use when:**
- You need memory safety + performance + no GC pauses (DBs, infra, games, real-time systems).
- You're writing CLI tooling for distribution (single-binary cargo output).
- You're targeting WebAssembly with non-trivial logic.
- You're building a library where memory bugs would be catastrophic (cryptography, networking primitives).

**Avoid when:**
- You're prototyping and need throwaway code velocity (Python, Go, TypeScript win).
- The team is small and the learning curve cost outweighs the safety benefit.
- Your problem is glue / orchestration / scripting — Rust's strengths are wasted.
- You depend on a runtime feature only available in GC'd languages (e.g., dynamic class loading).

## Alternatives

- [golang/go](../golang/go.md) — easier to learn, GC, less flexible, faster compile.
- **Zig** — manual memory management without a borrow checker, simpler than Rust, less mature ecosystem. See [mitchellh/ghostty](../mitchellh/ghostty.md) for a Zig case study.
- **C++ (modern)** — comparable performance, more flexible, no compile-time memory safety guarantees.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2012-01 | First public release at Mozilla. |
| 1.0 | 2015-05 | Stable, first edition (2015)[^1]. |
| 1.31 | 2018-12 | Edition 2018, NLL borrow checker. |
| 1.36 | 2019-07 | `Future` stable. |
| 1.39 | 2019-11 | `async`/`await` stable. |
| 1.56 | 2021-10 | Edition 2021. |
| 1.65 | 2022-11 | Generic associated types (GATs) stable. |
| 1.70 | 2023-06 | `OnceCell` / `OnceLock` stable. |
| 1.75 | 2023-12 | Async fn in traits stable. |
| 1.80 | 2024-07 | Lazy locks, exclusive range patterns. |
| 1.85 | 2025-02 | Edition 2024[^3]. |

## References

[^1]: Aaron Turon and Niko Matsakis, "Announcing Rust 1.0" — 2015-05-15. https://blog.rust-lang.org/2015/05/15/Rust-1.0.html
[^2]: Rust compiler dev guide. https://rustc-dev-guide.rust-lang.org/
[^3]: Rust Team, "Edition 2024". https://doc.rust-lang.org/edition-guide/rust-2024/
