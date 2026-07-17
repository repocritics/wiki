# async-rs/async-std

> An async mirror of the Rust standard library — now discontinued, with upstream pointing users to smol.

[GitHub repo](https://github.com/async-rs/async-std) ·
[Official website](https://async.rs) ·
[License: Apache-2.0](https://github.com/async-rs/async-std/blob/main/LICENSE-APACHE) (crate is dual MIT OR Apache-2.0)

## Overview

async-std is a Rust library that re-implements the shape of the standard library — `task`, `fs`, `net`, `io`, `sync`, `stream` — as `async`/`await` APIs[^1]. The premise was pedagogical as much as practical: if `std::fs::File::open` returns a `File`, then `async_std::fs::File::open` should be an `async fn` returning the same thing, so that developers already fluent in synchronous Rust could reach for async with minimal new surface to learn. It arrived in 2019 during the period when `async`/`await` was being stabilized in the language (Rust 1.39, late 2019) and competed directly with Tokio for mindshare as *the* async runtime.

The project has since been discontinued. The current README states plainly that async-std was created "to demonstrate the value of making a library as close to `std` as possible, but async," considers that demonstration successful, and now recommends that all users — and all libraries built on it — migrate to smol instead[^2]. The repository is not archived on GitHub and still receives occasional commits, but it is in maintenance-only status and should be treated as end-of-life for new projects.

The defining tension of async-std was familiarity versus ecosystem gravity. Its std-parity API was genuinely easier to onboard than Tokio's, but the async Rust ecosystem consolidated around Tokio for libraries (hyper, tonic, sqlx, and most networking crates target Tokio's traits), leaving async-std users to bridge runtimes or accept a smaller compatible-library set. That gravity, more than any technical defect, is why the project wound down.

## Getting Started

Note: this is documented for completeness. For new work, prefer smol or Tokio (see Alternatives).

```bash
cargo add async-std --features attributes
```

```rust
use async_std::task;

async fn say_hello() {
    println!("Hello, world!");
}

fn main() {
    task::block_on(say_hello())
}
```

With the `attributes` feature, an async main is available:

```rust
#[async_std::main]
async fn main() {
    let contents = async_std::fs::read_to_string("Cargo.toml").await.unwrap();
    println!("{contents}");
}
```

## Architecture / How It Works

async-std layered a std-shaped API over a multi-threaded work-stealing executor and a reactor for I/O readiness:

- **Executor / task spawning.** `task::spawn` schedules a future onto a global thread pool; `task::block_on` drives a future to completion on the current thread. Early versions used a bespoke single-allocation task and an adaptive executor that spawned blocking threads under load. Over time the internals were re-based onto the smol-rs building blocks — `async-executor`, `async-io`, and `async-global-executor` — so the runtime beneath the std-parity facade converged with smol's[^3].
- **Reactor.** Non-blocking I/O readiness is driven by `async-io`/`polling`, which wraps epoll/kqueue/IOCP. File-system calls, which are not natively non-blocking on most platforms, are offloaded to a blocking thread pool via `spawn_blocking` — the same pragmatic approach Tokio takes.
- **Traits.** async-std defined its own `Read`, `Write`, `Stream`, and `BufRead` async traits (largely aligned with `futures-io` / `futures-core`) rather than Tokio's `AsyncRead`/`AsyncWrite`. This trait split is the concrete reason async-std and Tokio libraries do not interoperate without adapters.
- **`attributes` feature.** Supplies `#[async_std::main]` and `#[async_std::test]` proc-macros so entry points and tests read like synchronous ones.

Because the public API mirrors `std` module-for-module, most "how do I do X" questions resolve by analogy: whatever the std path is, prefix `async_std::` and `.await` the call.

## Production Notes

- **End-of-life.** The single most important operational fact: upstream recommends migration to smol[^2]. New dependencies on async-std inherit a runtime that will not track future async language and ecosystem changes. Treat any greenfield use as technical debt from day one.
- **Ecosystem interop is the real cost.** The large majority of async networking and database crates (hyper, tonic, sqlx, redis, aws-sdk) are written against Tokio's `AsyncRead`/`AsyncWrite`. Using them from async-std requires `async-compat` or running a Tokio context alongside, which negates much of the simplicity that motivated the choice.
- **Runtime overlap with smol.** Because async-std later sat on top of smol-rs crates, a project that depends on both async-std and smol may pull two executors. Migration to smol is usually straightforward precisely because the substrate is shared.
- **`spawn_blocking` unboundedness.** Like other reactors, blocking work (file I/O, CPU-bound calls) is pushed to an auto-growing thread pool. Flooding it with long blocking calls can spawn a large number of OS threads; bound this at the application level.
- **Cancellation.** Dropping a future cancels it at the next await point. Code holding resources across awaits must be written to tolerate cancellation, the same discipline required under any Rust async runtime.

## When to Use / When Not

**Use when:**
- You are maintaining an existing async-std codebase and need reference material, not a rewrite yet.
- You are studying how an async std-parity API is designed — it remains a clean pedagogical example.

**Avoid when:**
- You are starting a new project — the upstream recommendation is smol, and the ecosystem default is Tokio.
- You need the broad library ecosystem (hyper/tonic/sqlx/etc.), which targets Tokio.
- You want a runtime with an active maintenance and security-response commitment.

## Alternatives

- smol-rs/smol — the upstream-recommended successor; small, modular async runtime built from the same `async-io`/`async-executor` crates. Use instead of async-std for new work that wants a lightweight runtime.
- tokio-rs/tokio — the de facto production runtime; use when you need the widest ecosystem of compatible networking/database crates.
- bytedance/monoio — thread-per-core, io_uring-based runtime; use when you need maximum single-node I/O throughput and can accept a Linux-first, less-portable model.
- async-rs/async-global-executor — the global-executor shim async-std itself used; use when you want async-std-style global spawning on top of smol primitives.
- futures-rs/futures — not a runtime but the trait/combinator layer; use for runtime-agnostic library code that lets the binary pick the executor.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-08-08 | Project started as `async`/`await` neared stabilization[^4]. |
| 1.0 | 2019-11 | 1.0 released around the Rust 1.39 async/await stabilization[^1]. |
| 1.6 | 2020 | Reactor/executor internals re-based onto smol-rs crates (`async-io`, `async-global-executor`)[^3]. |
| — | ~2023 | Development effectively winds down; README adds discontinuation notice recommending smol[^2]. |
| last push | 2025-08-15 | Occasional maintenance commits; not archived, but end-of-life. |

## References

[^1]: async.rs — project site and announcement material for the std-parity async library. https://async.rs
[^2]: async-std README, "async-std has been discontinued; use smol instead." https://github.com/async-rs/async-std/blob/main/README.md
[^3]: smol-rs organization — `async-io`, `async-executor`, `async-global-executor`, the shared substrate async-std adopted. https://github.com/smol-rs
[^4]: GitHub repository metadata, async-rs/async-std (created 2019-08-08). https://github.com/async-rs/async-std

## Tags

rust, async, async-await, runtime, concurrency, std-parity, discontinued, smol, tokio-alternative, executor
