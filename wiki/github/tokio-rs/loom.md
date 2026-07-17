# tokio-rs/loom

> Concurrency permutation testing for Rust — exhaustively explores thread interleavings and memory orderings under the C11 model to find data races and ordering bugs.

[GitHub repo](https://github.com/tokio-rs/loom) ·
[Documentation](https://docs.rs/loom) ·
[License: MIT](https://github.com/tokio-rs/loom/blob/master/LICENSE)

## Overview

Loom is a model checker for concurrent Rust. Instead of running a multithreaded test once and hoping the scheduler exposes a bug, loom runs the test closure many times, each time forcing a different legal interleaving of thread execution and a different legal outcome of relaxed-atomic operations under the C11 memory model[^1]. If any interleaving violates an assertion, deadlocks, or leaks, loom reports it — deterministically and reproducibly. It grew out of the Tokio project's own need to validate hand-written lock-free primitives, and today it is the standard verification tool for that class of code in the Rust ecosystem (Tokio, crossbeam, and many `Arc`/queue/lock implementations test with it).

The defining idea is that concurrency bugs are not random — they are specific interleavings that a normal test run almost never hits. A naive checker would try every interleaving and drown in combinatorial explosion. Loom borrows **dynamic partial-order reduction (DPOR)** techniques from the CDSChecker research checker[^2] to prune interleavings that are provably equivalent, making exhaustive checking of small tests tractable. The tradeoff that governs everything about using loom: the search space still grows explosively with the number of threads, atomic operations, and preemption points, so loom is a tool for tiny, focused unit tests of a single synchronization primitive — not for integration tests of a whole program.

Loom is not a linter or a sanitizer. It sees only the concurrency primitives you route through it. Code that reaches for real `std` atomics, raw `UnsafeCell`, or FFI is invisible to loom and gets no coverage.

## Getting Started

Loom is a dev-dependency gated behind a `cfg(loom)` flag so it never affects release builds:

```toml
# Cargo.toml
[target.'cfg(loom)'.dependencies]
loom = "0.7"
```

In the code under test, conditionally swap `std` types for loom's instrumented equivalents:

```rust
#[cfg(loom)]
use loom::sync::atomic::AtomicUsize;
#[cfg(not(loom))]
use std::sync::atomic::AtomicUsize;
```

Write a test inside `loom::model`, which drives the exhaustive exploration:

```rust
use loom::sync::Arc;
use loom::sync::atomic::AtomicUsize;
use loom::sync::atomic::Ordering::{Acquire, Release, Relaxed};
use loom::thread;

#[test]
#[should_panic] // this interleaving-sensitive code is intentionally buggy
fn buggy_concurrent_inc() {
    loom::model(|| {
        let num = Arc::new(AtomicUsize::new(0));
        let ths: Vec<_> = (0..2).map(|_| {
            let num = num.clone();
            thread::spawn(move || {
                let curr = num.load(Acquire);
                num.store(curr + 1, Release); // non-atomic read-modify-write
            })
        }).collect();
        for th in ths { th.join().unwrap(); }
        assert_eq!(2, num.load(Relaxed));
    });
}
```

Loom code only compiles when the flag is set, so run the test with:

```console
RUSTFLAGS="--cfg loom" cargo test --test buggy_concurrent_inc --release
```

## Architecture / How It Works

Loom provides shadow implementations of the `std` concurrency surface: `loom::sync` (`Arc`, `Mutex`, `RwLock`, `Condvar`, `atomic::*`), `loom::thread`, `loom::cell::UnsafeCell`, and `loom::lazy_static`. These are not thin wrappers — each operation registers with loom's internal scheduler so the checker knows where a thread can be preempted and what values an atomic load is permitted to return.

`loom::model(closure)` runs the closure repeatedly. On each run, loom's cooperative scheduler chooses one path through the tree of possible interleavings: at every scheduling point (spawn, atomic access, lock, yield) it decides which thread proceeds and, for reads of relaxed atomics, which previously-written value becomes visible per the C11 happens-before relation. After a run completes, loom backtracks and picks the next unexplored branch. DPOR prunes branches whose reordering cannot change the observable outcome, so independent operations are not re-permuted against each other[^2].

Loom's `UnsafeCell` is the mechanism for catching data races in `unsafe` code: `with` / `with_mut` accessors record every read and write, and loom flags concurrent unsynchronized access. It also tracks object lifetimes — if an `Arc` or allocation is still live when the model ends, loom reports a leak, which surfaces reference-count bugs. Deadlocks appear as interleavings where all threads are blocked.

Because the scheduler is deterministic, a failing interleaving is fully reproducible, and loom prints the sequence of thread steps that led to it. This is the payoff over stress-testing: a red test is a concrete, replayable counterexample rather than a flaky failure.

## Production Notes

**State explosion is the whole game.** The number of interleavings grows factorially with threads and roughly exponentially with the count of atomic/lock operations. A test with 3 threads each doing a handful of atomic ops can already take minutes or fail to terminate. The primary control is `LOOM_MAX_PREEMPTIONS` (commonly set to 2 or 3): it bounds how many times the scheduler may preempt a running thread, trading exhaustiveness for a tractable, high-value subset of the search space. Most real loom suites run bounded, not exhaustive.

**Keep tests minimal.** The practical discipline is one primitive, the fewest threads that can trigger the bug (usually 2), and the shortest operation sequence. Loop counts that are fine in a normal test (`0..100`) will make loom hang. Reduce constants aggressively under `cfg(loom)`.

**Checkpointing for long runs.** `LOOM_CHECKPOINT_FILE` plus `LOOM_CHECKPOINT_INTERVAL` let loom persist progress and resume, so a large exploration that finds a failure can be re-run straight to the failing iteration instead of from the start. `LOOM_LOG` enables execution logging to inspect the interleaving.

**It is not a complete model of C11.** Per the project's own documentation, loom does not implement the full memory model. `SeqCst` accesses are treated as `AcqRel`, which is weaker — this can produce false alarms on code that genuinely relies on `SeqCst` (see upstream issue #180); note that `fence(SeqCst)` is handled. Conversely, loom does not explore certain load-buffering behaviors that C11 permits, so it can miss real bugs (unsound in that direction). Treat a green loom run as strong evidence, not a proof.

**Coverage is limited to instrumented types.** Anything using `std` atomics directly, raw pointers, or external C code is not seen by loom. The `cfg(loom)` swap has to be threaded through every primitive you want checked, which is intrusive in existing code and a common source of "why didn't loom catch this."

**Cost and cadence.** Loom tests are slow and typically run in a dedicated, less-frequent CI job (often `--release`, since debug is slower still) rather than on every push. The repo itself is mature and low-churn — actively maintained by the Tokio team but evolving slowly, with a substantial open-issue backlog reflecting known model-completeness gaps rather than breakage.

## When to Use / When Not

**Use when:**
- You are writing or reviewing lock-free / low-level `unsafe` concurrency: a custom queue, allocator, `Arc`-like refcount, lock, or channel.
- You need a reproducible counterexample for an ordering bug that stress tests only hit occasionally.
- You want confidence that relaxed/acquire/release orderings on a small primitive are actually correct.

**Avoid when:**
- You are testing high-level application logic or anything with many threads / long loops — the search space is intractable.
- Your concurrency is ordinary `Mutex`-guarded shared state with no `unsafe`; the value over normal tests is low.
- You need a soundness guarantee: loom's model is incomplete (SeqCst, load buffering) and is a bug-finder, not a verifier.
- The code can't be refactored to route its primitives through `loom::sync` under `cfg(loom)`.

## Alternatives

- crossbeam-rs/crossbeam — not a checker, but its lock-free structures are themselves loom-tested; a source of vetted primitives if you'd rather not hand-roll and verify your own.
- rust-lang/miri — interpreter that detects undefined behavior and some data races in `unsafe` code; explores far fewer interleavings than loom but needs no type swapping and catches a broader class of UB. Use miri for general `unsafe` correctness, loom for exhaustive ordering search.
- Shuttle (from AWS Labs) — randomized concurrency testing for Rust that scales to larger tests by sampling interleavings instead of exhausting them; use when loom's state explosion makes exhaustive checking impossible.
- ThreadSanitizer (via `-Z sanitizer=thread`) — runtime race detector; finds races that actually occur during a run, without loom's exhaustive scheduling. Use it for realistic workloads, loom for adversarial scheduling of a primitive.
- CDSChecker (C/C++) — the research checker whose reduction techniques loom draws on; relevant if you work in C/C++ rather than Rust.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-12 | Repository created; extracted from Tokio's internal concurrency-testing needs[^3]. |
| 0.x | 2019–2021 | Iteration on the C11 model, DPOR, and the `cfg(loom)` type-swap workflow. |
| 0.7 | current | Latest published series on crates.io; API stable, slow evolution[^4]. |

Exact per-release dates are not restated here to avoid inaccuracy; consult the crates.io release history for authoritative version dates.

## References

[^1]: C++ / C11 memory ordering (the model loom approximates). https://en.cppreference.com/w/cpp/atomic/memory_order
[^2]: Brian Norris and Brian Demsky, "CDSChecker: Checking Concurrent Data Structures Written with C/C++ Atomics" — the state-reduction techniques loom's README credits. http://plrg.eecs.uci.edu/publications/toplas16.pdf
[^3]: tokio-rs/loom repository, created 2018-12-15. https://github.com/tokio-rs/loom
[^4]: loom on crates.io. https://crates.io/crates/loom

## Tags

rust, concurrency, testing, model-checking, memory-model, atomics, lock-free, unsafe, tokio, verification, dpor, c11
