# rayon-rs/rayon

> Data-parallelism library for Rust — turn `iter()` into `par_iter()` and get work-stealing multithreading with compile-time data-race freedom.

[GitHub repo](https://github.com/rayon-rs/rayon) ·
[docs.rs](https://docs.rs/rayon) ·
[License: MIT OR Apache-2.0](https://github.com/rayon-rs/rayon/blob/main/README.md#license)

## Overview

Rayon is a library for CPU-bound data parallelism in Rust. Its central promise is that converting a sequential computation into a parallel one is usually a one-token change — `foo.iter()` becomes `foo.par_iter()` — and the rest of the pipeline (`map`, `filter`, `sum`, `collect`) keeps working[^1]. The scheduler decides at runtime how to split the work across a thread pool, so the caller never manages threads directly. It was introduced by Niko Matsakis in late 2015 as a demonstration that Rust's ownership model makes safe, ergonomic parallelism tractable[^2].

The defining property is that Rayon leans entirely on Rust's `Send`/`Sync` type system for correctness. If a parallel closure would introduce a data race, the program does not compile — there is no runtime race detector because the guarantee is static. In practice this means "if it compiles, it computes the same result as the sequential version," with the important caveat that side effects (channel sends, disk writes, logging) may be reordered or interleaved[^1].

The tension to understand before adopting it: Rayon is a **fork-join, CPU-work scheduler, not a general concurrency runtime.** It expends its parallelism on dividing compute-heavy loops, and it assumes worker threads make forward progress. Blocking a worker (I/O, a contended mutex, sleeping, or awaiting an async future) starves the shared pool and, in recursive cases, can deadlock. Rayon and an async runtime like Tokio solve different problems and are frequently used together in the same process, each owning its own thread pool.

## Getting Started

```toml
# Cargo.toml
[dependencies]
rayon = "1.12"
```

```rust
use rayon::prelude::*;   // brings the parallel-iterator traits into scope

fn sum_of_squares(input: &[i32]) -> i32 {
    input.par_iter()        // <-- the only change from a sequential version
         .map(|&i| i * i)
         .sum()
}
```

For explicit fork-join outside the iterator API, use `join` (two tasks) or `scope` (dynamic fan-out):

```rust
// Parallel quicksort: recurse on both halves, potentially in parallel.
fn quick_sort<T: Send + Ord>(v: &mut [T]) {
    if v.len() <= 1 { return; }
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    rayon::join(|| quick_sort(lo), || quick_sort(hi));
}
```

Rayon requires `rustc 1.85.0` or newer[^1].

## Architecture / How It Works

Rayon is two crates. **`rayon-core`** holds the scheduler: the global thread pool, the work-stealing deques, and the primitives `join`, `scope`, `spawn`, and `ThreadPool`. **`rayon`** builds the parallel-iterator layer on top. `rayon-core` is deliberately kept at a single semver-compatible version across the ecosystem so that every dependency shares **one** global thread pool rather than each spawning its own[^3].

**Work stealing.** The core primitive `join(a, b)` expresses *potential* parallelism, not guaranteed parallelism. When you call `join`, Rayon pushes closure `b` onto the current worker's local deque and starts running `a` immediately. If another worker is idle, it may *steal* `b` and run it concurrently; if no one steals it, the same thread runs `b` after `a` finishes. This means the cost of `join` when the pool is saturated collapses to little more than a couple of function calls — parallelism is opportunistic and adapts to available cores. The deques are the Chase–Lev work-stealing design (via `crossbeam-deque`).

**Parallel iterators** are a producer/consumer bridge, not a thin wrapper over `join`. A `ParallelIterator` describes work; the `plumbing` layer (`Producer`, `Consumer`, `Folder`, `bridge`) recursively splits the input in half, running each half under `join`, until pieces are small enough to process sequentially. Splitting is driven by both data length and a heuristic tied to the number of active threads, so granularity self-tunes rather than being fixed. Iterators that carry an exact length implement `IndexedParallelIterator`, which unlocks order-dependent combinators like `zip`, `enumerate`, and indexed `collect`.

**The default pool** is created lazily on first use and sized to the number of logical CPUs (hyperthreads included). It can be overridden by `RAYON_NUM_THREADS` or, programmatically, by `ThreadPoolBuilder::build_global()` — but only **once**, and only before any Rayon API is touched. For isolation, `ThreadPoolBuilder::build()` yields a private `ThreadPool`, and `pool.install(|| ...)` runs a closure whose Rayon calls route to that pool instead of the global one.

## Production Notes

**Never block a worker thread.** This is the dominant footgun. Rayon's threads are meant to churn CPU work; if a task performs blocking I/O, sleeps, or awaits an async future, it parks a worker that the scheduler assumed was making progress. Recursive `join`/`scope` code that blocks on a resource held by another Rayon task can deadlock the entire pool. Keep I/O and `.await` out of Rayon closures; hand blocking work to a dedicated pool (e.g. `tokio::task::spawn_blocking`) instead.

**The global pool is process-wide and shared.** Every crate in your dependency tree that uses Rayon (polars, ndarray, image, and many others) draws from the same default pool. Two libraries each assuming they own all cores can oversubscribe. If you need a bounded thread count, call `build_global()` early in `main`, before any dependency touches Rayon — a second call, or a call after first use, returns an error.

**Small workloads can be slower parallel than sequential.** `par_iter` has real per-split and scheduling overhead. For short iterators or cheap per-item work, the coordination cost dominates and a plain `iter()` wins. When items are uneven or fine-grained, tune granularity with `with_min_len` / `with_max_len` rather than letting the splitter over-divide.

**Panics propagate, but abort the whole computation.** If one closure panics, Rayon stops the parallel operation and re-raises the panic on the caller's thread after in-flight tasks unwind. Partial side effects already committed by other items are not rolled back, so non-idempotent effects inside a parallel loop need care.

**Ordering and determinism.** Results of a parallel iterator match the sequential order for order-preserving combinators, but *side effects* (prints, channel sends, file writes) occur in scheduling order, which is nondeterministic. Reductions must be associative to be correct under arbitrary split points.

**WebAssembly.** On `wasm32` targets without threads, Rayon silently falls back to sequential execution — code compiles and runs, just on one core. Real browser multithreading requires the external `wasm-bindgen-rayon` adapter plus cross-origin isolation headers and a nightly-ish build setup[^1].

## When to Use / When Not

**Use when:**
- You have a CPU-bound loop or divide-and-conquer computation over in-memory data and want it to scale across cores with minimal code change.
- The work is embarrassingly parallel or cleanly recursive (map/reduce, sorting, image processing, numerical kernels, batch transforms).
- You want data-race safety guaranteed by the compiler rather than by discipline.

**Avoid when:**
- The workload is I/O-bound or involves `.await` — that is async concurrency (Tokio/async-std), not data parallelism.
- Per-item work is tiny and the collection is small; scheduling overhead will outweigh the gains.
- You need fine manual control over thread lifetimes, affinity, or a custom scheduler — reach for lower-level primitives.
- You are targeting single-core WASM in the browser without the ability to add the adapter and isolation headers.

## Alternatives

- tokio-rs/tokio — use instead when the problem is I/O concurrency (network, timers, many sockets), not CPU-bound compute; the two often coexist in one process.
- crossbeam-rs/crossbeam — use when you want the building blocks (scoped threads, channels, work-stealing deques) to assemble your own scheduling rather than Rayon's fork-join model.
- rust-lang/rust — use `std::thread::scope` directly when parallelism is coarse and static (a handful of long-lived threads) and a work-stealing pool is overkill.
- rust-lang/portable-simd — use when the parallelism fits *within* a core via SIMD lanes instead of *across* cores via threads; complementary to Rayon, not a replacement.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-12 | Introduced by Niko Matsakis; `join`-based work stealing[^2]. |
| rayon-core split | 2017 | Scheduler extracted into `rayon-core` so the ecosystem shares one global pool[^3]. |
| 1.0 | 2018 | API stabilized; parallel-iterator interface committed to semver. |
| 1.5 | 2021 | `try_fold`/`try_reduce` and short-circuiting refinements. |
| 1.10–1.12 | 2024–2026 | Ongoing releases; MSRV raised to Rust 1.85[^1]. |

## References

[^1]: Rayon README and usage notes (crates.io `rayon` 1.12, MSRV 1.85, WASM fallback). https://github.com/rayon-rs/rayon/blob/main/README.md
[^2]: Niko Matsakis, "Rayon: data parallelism in Rust" — 2015-12-18. https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/
[^3]: `rayon-core` crate — the shared scheduler underpinning the single global thread pool. https://docs.rs/rayon-core

## Tags

rust, parallelism, concurrency, data-parallelism, work-stealing, multithreading, thread-pool, parallel-iterator, performance, library
