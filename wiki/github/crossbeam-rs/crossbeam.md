# crossbeam-rs/crossbeam

> The foundational toolbox of lock-free and low-level concurrency primitives for Rust — channels, work-stealing deques, epoch-based reclamation, and scoped threads.

[GitHub repo](https://github.com/crossbeam-rs/crossbeam) ·
[docs.rs](https://docs.rs/crossbeam) ·
[License: Apache-2.0 OR MIT](https://github.com/crossbeam-rs/crossbeam#license)

## Overview

Crossbeam is a collection of concurrency primitives for Rust, maintained by the
`crossbeam-rs` organization since 2015[^1]. It sits one layer below the ergonomic
parallelism crates: where Rayon gives you parallel iterators and Tokio gives you
an async runtime, Crossbeam gives you the building blocks — lock-free queues, a
memory-reclamation scheme, cache-line padding, and channels — that those higher
crates are built on. Rayon's task scheduler, for example, is built on
`crossbeam-deque`[^2].

The project's defining tension is that it lives in the gap between "unsafe atomics"
and "the standard library caught up." Several Crossbeam features were later
absorbed into `std`: scoped threads landed in `std::thread::scope` in Rust 1.63[^3],
and the standard library's `mpsc` channel was reimplemented on top of
`crossbeam-channel`'s algorithm in Rust 1.67[^4]. What remains in Crossbeam is the
material `std` does not want to commit to a stable API — epoch-based garbage
collection, work-stealing deques, MPMC channels with `select!`, and `no_std`-capable
atomics. That makes Crossbeam simultaneously essential infrastructure and a crate
whose surface area slowly erodes as `std` grows.

Crossbeam is a facade crate: `crossbeam` itself mostly re-exports smaller subcrates
(`crossbeam-channel`, `crossbeam-deque`, `crossbeam-epoch`, `crossbeam-queue`,
`crossbeam-utils`), each independently versioned and publishable. Most production
users depend on a single subcrate rather than the umbrella.

## Getting Started

```toml
[dependencies]
crossbeam = "0.8"
```

```rust
use crossbeam::channel;
use crossbeam::thread;

fn main() {
    let (tx, rx) = channel::unbounded();

    // Scoped threads can borrow stack locals without 'static or Arc.
    thread::scope(|s| {
        for i in 0..4 {
            let tx = tx.clone();
            s.spawn(move |_| tx.send(i * i).unwrap());
        }
    })
    .unwrap();
    drop(tx); // close the channel so the receiver loop terminates

    let sum: i32 = rx.iter().sum();
    println!("{sum}"); // 14
}
```

The `select!` macro is the reason many teams reach for `crossbeam-channel` over
`std::sync::mpsc`: it lets a single thread wait on multiple channels (plus timeouts)
in one blocking call, which the standard channel cannot express.

## Architecture / How It Works

Each subcrate solves one hard problem:

- **`crossbeam-epoch`** implements epoch-based reclamation (EBR), a technique for
  freeing memory in lock-free data structures without hazard pointers or reference
  counting. Readers "pin" the current epoch; memory retired while a reader is pinned
  is not freed until all pinned readers advance. This is the load-bearing subcrate —
  it is how you can safely `free` a node that another thread might still be reading.
  It is also the hardest to use correctly and the source of most of Crossbeam's
  `unsafe`.
- **`crossbeam-deque`** is a Chase-Lev work-stealing deque: one owner pushes/pops
  from one end lock-free, other threads steal from the other end. It exists almost
  entirely to serve task schedulers like Rayon[^2].
- **`crossbeam-channel`** provides MPMC channels (bounded, unbounded, rendezvous)
  plus `select!`. Its internals were considered good enough that `std` adopted the
  algorithm for its own `mpsc`[^4].
- **`crossbeam-queue`** exposes `ArrayQueue` (bounded, fixed buffer) and `SegQueue`
  (unbounded, grows in segments) — concrete lock-free queues built on the epoch GC.
- **`crossbeam-utils`** holds the widely-used small tools: `AtomicCell`, `CachePadded`,
  `Backoff`, and the scoped-thread implementation.

A subtle detail worth knowing: `AtomicCell<T>` is only lock-free when `T` fits a
native atomic width. For larger `T` it silently falls back to a sharded spinlock
keyed on the value's address — correct, but not wait-free and not what the name
suggests. Check `AtomicCell::<T>::is_lock_free()` if it matters.

## Production Notes

- **Dropped senders/receivers change semantics.** `crossbeam-channel` only signals
  disconnection once *all* clones of the opposite endpoint are dropped. A `recv()`
  loop that never terminates is almost always a leaked `Sender` clone somewhere.
- **`select!` is a macro, not zero-cost dispatch.** It registers with every channel's
  internal wait queue on each iteration; in tight hot loops with many cases it has
  measurable overhead versus a hand-rolled poll. It also cannot select over a runtime
  `Vec` of channels — use `Select` (the builder API) for dynamic sets.
- **Epoch GC defers, it does not collect promptly.** Retired memory is reclaimed only
  when the global epoch advances, which requires threads to keep pinning. A workload
  that pins once and then goes idle can hold retired garbage indefinitely; memory usage
  can look like a leak under bursty load.
- **`no_std` support is partial and feature-gated.** `AtomicCell`, `CachePadded`,
  `Backoff`, and `AtomicConsume` work without `std`; the queues and epoch GC require
  the `alloc` feature; channels and scoped threads require `std`.
- **Prefer `std` where it now suffices.** If you only need scoped threads
  (`std::thread::scope`, since 1.63) or a simple channel, the standard library removes
  a dependency. Reach for Crossbeam when you specifically need `select!`, MPMC
  fan-out, work-stealing, or lock-free queues.
- **MSRV moves with releases.** The minimum supported Rust version is 1.74; the project
  policy is to support stable back at least one year and bump the minor version each
  time the MSRV rises[^5]. Pin accordingly in libraries with conservative MSRV
  requirements.

## When to Use / When Not

**Use when:**
- You are building a lock-free data structure and need a real memory-reclamation
  scheme (`crossbeam-epoch`).
- You need MPMC channels, `select!` over multiple channels, or bounded/rendezvous
  semantics that `std::sync::mpsc` cannot express.
- You are writing a task scheduler or thread pool and want a proven work-stealing deque.
- You need `CachePadded` / `Backoff` to hand-tune contention in low-level code.

**Avoid when:**
- You only need scoped threads or a basic channel — the standard library now covers
  both with zero dependencies.
- You want async message passing — use `tokio::sync::mpsc` or a runtime-native channel;
  Crossbeam channels are blocking.
- You want high-level data parallelism — use Rayon, which sits above Crossbeam and hides
  these primitives.

## Alternatives

- rust-lang/rust — `std::thread::scope` and `std::sync::mpsc` now cover the two most
  commonly borrowed Crossbeam features with no external dependency; use them when that
  is all you need.
- rayon-rs/rayon — data-parallel iterators built on top of `crossbeam-deque`; use it
  when you want parallelism, not primitives.
- tokio-rs/tokio — use `tokio::sync::mpsc`/`broadcast` when your concurrency is async
  rather than OS-thread-based.
- zesterer/flume — pure-Rust MPMC channel with both sync and async APIs and a lighter
  dependency tree; use it as a `crossbeam-channel` alternative when you also need async.
- ibraheemdev/seize — a modern, arguably simpler memory-reclamation scheme; use it
  instead of `crossbeam-epoch` when you want reclamation without the epoch model.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial crate | 2015-05 | First `crossbeam` release; original scoped-thread API[^1]. |
| — | 2015–2016 | Scoped threads redesigned after the `mem::forget` soundness issue[^6]. |
| 0.8.0 | 2020-12 | Workspace split into independently versioned subcrates; `no_std` support. |
| (std) | 2022-08 | `std::thread::scope` stabilized in Rust 1.63, echoing Crossbeam's API[^3]. |
| (std) | 2023-01 | `std::sync::mpsc` reimplemented on `crossbeam-channel` in Rust 1.67[^4]. |
| 0.8.x | 2026 | Ongoing maintenance releases; MSRV raised to 1.74[^5]. |

## References

[^1]: crossbeam-rs/crossbeam — repository created 2015-05-13. https://github.com/crossbeam-rs/crossbeam
[^2]: Rayon depends on `crossbeam-deque` for its work-stealing scheduler. https://github.com/rayon-rs/rayon
[^3]: Rust 1.63.0 release notes — stabilized `std::thread::scope`. https://blog.rust-lang.org/2022/08/11/Rust-1.63.0.html
[^4]: Rust 1.67.0 release notes — `std::sync::mpsc` reimplemented on top of `crossbeam-channel`. https://blog.rust-lang.org/2023/01/26/Rust-1.67.0.html
[^5]: Crossbeam README, "Compatibility" — MSRV 1.74, minor bump on each MSRV increase. https://github.com/crossbeam-rs/crossbeam#compatibility
[^6]: Rust issue #24292 — scoped threads and the `mem::forget` soundness problem that forced the API redesign. https://github.com/rust-lang/rust/issues/24292

## Tags

rust, concurrency, lock-free, channels, epoch-gc, work-stealing, data-structures, synchronization, no-std, parallelism
