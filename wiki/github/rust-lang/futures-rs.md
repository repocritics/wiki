# rust-lang/futures-rs

> The foundational async utility crate for Rust — traits like `Stream` and `Sink` that `std` never adopted, plus the combinators and executors around `std::future::Future`.

[GitHub repo](https://github.com/rust-lang/futures-rs) ·
[Official website](https://rust-lang.github.io/futures-rs/) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/futures-rs/blob/master/LICENSE-APACHE)

## Overview

`futures-rs` is the async-foundations crate maintained under the Rust project itself[^1]. In the pre-async/await era it *was* async Rust: the original `Future` trait, the `Stream`/`Sink` traits, and a large combinator library all lived here (the 0.1 line, from 2016). When `async`/`await` and `std::future::Future` were stabilized in 2019, the core `Future` trait moved into the standard library and `futures-rs` was rewritten as the 0.3 line — a companion crate that supplies everything `std` deliberately left out[^2].

That "everything std left out" is the defining tension. As of 2026, `std` has `Future` and the `async`/`await` syntax but still ships no `Stream` (async iterator), no `Sink`, and no async I/O traits. `futures-rs` fills those gaps via `futures-core` (`Stream`), `futures-sink` (`Sink`), and `futures-io` (`AsyncRead`/`AsyncWrite`), plus the extension-trait combinators (`StreamExt`, `TryStreamExt`, `SinkExt`) and the `join!`/`select!` macros. It is not a runtime — it ships only a minimal single-threaded executor for tests and examples. Real programs pair it with Tokio, async-std, or smol.

The crate has stayed at 0.3.x for years on purpose. Its traits are candidate designs for eventual `std` inclusion, and bumping to 1.0 would freeze APIs the language team may still change (the nightly `AsyncIterator`/`async_iter` work is the ongoing attempt to land `Stream` in `std`). The result is a crate that is simultaneously extremely stable in practice and formally "pre-1.0."

## Getting Started

```toml
[dependencies]
futures = "0.3"
```

For bare-metal / `no_std` (reduced API surface, no executor, no `std`-backed channels):

```toml
[dependencies]
futures = { version = "0.3", default-features = false }
```

The current release requires Rust 1.71 or later[^3].

```rust
use futures::stream::{self, StreamExt};

fn main() {
    // block_on is the built-in minimal executor — fine for examples, not for servers.
    futures::executor::block_on(async {
        let sum: u32 = stream::iter(1..=5)
            .map(|n| n * 2)
            .filter(|n| futures::future::ready(n % 3 != 0))
            .fold(0, |acc, n| async move { acc + n })
            .await;
        println!("{sum}"); // 24  (2,4,8,10 — 6 filtered out)
    });
}
```

```rust
use futures::{future, join, select, FutureExt};

async fn work() {
    // join! awaits concurrently on one task; select! races.
    let (a, b) = join!(future::ready(1), future::ready(2));

    let mut f1 = future::ready(10).fuse();
    let mut f2 = future::pending::<i32>().fuse();
    let first = select! { x = f1 => x, y = f2 => y };
    println!("{} {} {}", a, b, first);
}
```

## Architecture / How It Works

The public `futures` crate is a thin umbrella that re-exports a set of small sub-crates, each independently versioned in-tree:

- **`futures-core`** — the trait definitions with no dependencies: `Stream`, `TryStream`, `FusedStream`, and the re-exported `Future`. Libraries that only need to *name* a `Stream` in their public API depend on this, not the full `futures`.
- **`futures-sink`** — the `Sink` trait (the write-side dual of `Stream`), kept separate for the same reason.
- **`futures-io`** — `AsyncRead`, `AsyncWrite`, `AsyncSeek`, `AsyncBufRead`.
- **`futures-channel`** — `oneshot` and `mpsc` async channels.
- **`futures-task`** — `Waker`/`ArcWake` utilities for hand-writing futures.
- **`futures-util`** — the bulk: all the combinator extension traits (`FutureExt`, `StreamExt`, `SinkExt`, `TryFutureExt`, `TryStreamExt`), `FuturesUnordered`, `select!`/`join!` machinery.
- **`futures-executor`** — `block_on`, `LocalPool`, `ThreadPool`.
- **`futures-macro`** — proc-macros backing `join!`/`select!`.

Combinators are **zero-cost** in the specific sense that each adapter (`.map`, `.filter`, `.then`) is a distinct type wrapping the inner future/stream; chaining them builds a compile-time state machine rather than a boxed callback chain, and the whole thing lowers to a `poll` loop with no per-item heap allocation. The cost you *do* pay is type-level: deeply chained adapters produce enormous types and slow compiles, and returning them from a function usually forces `impl Stream<Item = ...>` or a `Box<dyn Stream>`.

`FuturesUnordered` (and `stream::FuturesOrdered`, `select_all`) is the primitive for driving many futures concurrently on a single task. It polls only the sub-futures whose wakers fired, which is efficient but is also the crate's most common footgun (see below).

## Production Notes

**The two-`AsyncRead` split.** `futures::io::AsyncRead`/`AsyncWrite` and `tokio::io::AsyncRead`/`AsyncWrite` are *different, incompatible traits* with different method signatures. Most of the networking ecosystem is built on Tokio's versions. Bridging requires the `Compat`/`TokioAsyncReadCompatExt` adapters from `tokio-util`'s `compat` module. This surprises nearly everyone once. Decide early which trait family a library targets and state it.

**`Stream` is not `std`.** Because the trait lives in `futures-core`, exposing a `Stream` in a public API pulls a `futures` dependency into your consumers, and there is no language-level `for await` loop — you iterate with `while let Some(x) = stream.next().await`. Libraries that want to avoid the dependency sometimes hand-roll `poll_next` against `futures-core` alone.

**`FuturesUnordered` starvation and cancellation.** A future stored in `FuturesUnordered` is only driven while the outer stream is being polled; if you `select!` between it and a fast branch that always wins, its members can starve. Dropping the set drops all in-flight futures at their current await point — correct only if those futures are cancel-safe. Unbounded `FuturesUnordered` is also a memory-growth vector if producers outrun completion.

**`select!` requires `FusedFuture`.** Branches must be fused (`.fuse()`) or already `FusedFuture`, otherwise a completed branch can be polled again and panic. `select!` also moves/borrows its operands in ways that fight the borrow checker; `pin_mut!` on the futures first is the standard fix. Default selection is pseudo-random for fairness; use `select_biased!` for top-to-bottom priority.

**The executor is a stub.** `futures::executor::block_on`/`ThreadPool` exist for examples, tests, and simple CLIs. They have no timer, no I/O reactor, and no work-stealing tuned for load. Anything doing real network or timer work wants Tokio (or async-std/smol). Do not benchmark "async Rust" against the built-in executor.

**Version stability.** 0.3.x has held API compatibility for years; upgrades within 0.3 are routine. The real churn is MSRV bumps (now 1.71) and the occasional deprecation of a niche combinator. The `unstable`/`bilock`/`write-all-vectored` feature flags gate APIs that may still change.

## When to Use / When Not

**Use when:**
- You need the `Stream` or `Sink` trait, or their combinators, in library code.
- You're writing runtime-agnostic async code and want to depend on traits, not on Tokio.
- You're hand-implementing a future/stream and need `Waker`/`ArcWake` helpers or `join!`/`select!`.
- You want a dependency-light `no_std` async trait surface for embedded work.

**Avoid when:**
- You only need `async fn` and `.await` — those are in `std`; you may not need the crate at all.
- You want a production async runtime — that's Tokio/async-std/smol, not `futures-executor`.
- Your stack is all-Tokio and you can use `tokio-stream` + `tokio::io` traits directly, avoiding the `AsyncRead` split.
- You're on a tight compile-time budget and would be pulling in `futures-util` just for one combinator (depend on `futures-core` alone instead).

## Alternatives

- tokio-rs/tokio — the dominant runtime; `tokio-stream` and `tokio-util` cover most combinator needs without `futures-util`. Use instead when your whole app is already on Tokio.
- async-rs/async-std — `std`-mirroring async API with a bundled runtime. Use when you want a familiar std-shaped surface over raw traits (note: largely in maintenance mode).
- smol-rs/smol — small runtime built on `async-io`/`async-executor`. Use when you want a minimal, auditable executor rather than the `futures` stub.
- tokio-rs/async-stream — `stream!`/`try_stream!` macros to write streams with `yield`. Use instead of chaining `StreamExt` combinators when the logic reads better imperatively.
- rust-lang `std` (`async`/`await`) — use alone when you never touch `Stream`/`Sink` and don't need combinators.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2016-08 | Original combinator library; its own `Future`/`Stream`/`Sink` traits with a built-in error type[^1]. |
| 0.2.0 | 2018-04 | Short-lived transitional `task::Context` rework; superseded quickly. |
| (std) `Future` | 2019-07 | `Future` trait stabilized into `std`/`core` in Rust 1.36. |
| (std) `async`/`await` | 2019-11 | `async`/`await` syntax stabilized in Rust 1.39. |
| 0.3.0 | 2019-11 | Full rewrite atop `std::future::Future` + `Pin`; today's API shape[^2]. |
| 0.3.x | 2020–2026 | Long-running stable line; incremental combinators, MSRV moved to 1.71[^3]. |

## References

[^1]: Aaron Turon, "Zero-cost futures in Rust" — 2016-08-11. https://aturon.github.io/blog/2016/08/11/futures/
[^2]: futures-rs 0.3.0 release notes. https://github.com/rust-lang/futures-rs/releases/tag/0.3.0
[^3]: futures-rs README (MSRV, feature flags, dual license). https://github.com/rust-lang/futures-rs#readme

## Tags

rust, async, futures, stream, concurrency, async-await, no-std, systems-programming, async-foundations
