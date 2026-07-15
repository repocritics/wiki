# Amanieu/parking_lot

> Compact, adaptive synchronization primitives for Rust — a WebKit-style parking lot instead of OS mutexes.

[GitHub repo](https://github.com/Amanieu/parking_lot) ·
[Docs (parking_lot)](https://docs.rs/parking_lot/) ·
License: Apache-2.0 OR MIT

## Overview

parking_lot provides drop-in replacements for the Rust standard library's `Mutex`, `RwLock`, `Condvar`, and `Once`, plus a `ReentrantMutex` that std does not offer. Its central design idea, borrowed from WebKit's `WTF::ParkingLot`[^1], is to keep the lock objects themselves tiny (a `Mutex` is one byte; `RwLock` and `Condvar` are one word) and move all the expensive queueing-and-suspending machinery out to a global side table keyed by lock address. A lock only pays for a queue when it is actually contended.

The library predates and heavily influenced the modern Rust standard library. When it was written, `std::sync::Mutex` on many platforms wrapped a pthread mutex and required a heap allocation; parking_lot's inline, allocation-free locks were substantially smaller and faster. The README reports `parking_lot::Mutex` as roughly 1.5x faster than std uncontended and up to 5x faster under contention on x86_64 Linux[^2]. Those numbers were measured against an older std; see Production Notes for why the gap has narrowed.

The defining tradeoff is philosophical: parking_lot deliberately drops std's *lock poisoning*. If a thread panics while holding a `parking_lot::Mutex`, the lock is simply released and the next acquirer sees no error. This removes the `Result`/`unwrap()` ceremony that pervades std locking, but it also removes the built-in signal that shared state may be inconsistent after a panic. Choosing parking_lot is partly a choice to opt out of that safety rail.

## Getting Started

```toml
# Cargo.toml
[dependencies]
parking_lot = "0.12"
```

```rust
use parking_lot::Mutex;
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = Vec::new();

    for _ in 0..8 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            // lock() returns the guard directly — no Result, no poisoning.
            let mut n = c.lock();
            *n += 1;
        }));
    }

    for h in handles { h.join().unwrap(); }
    assert_eq!(*counter.lock(), 8);
}
```

`Mutex::new` is a `const fn`, so locks can live in `static` items without lazy initialization.

## Architecture / How It Works

parking_lot is three crates layered on top of each other:

1. **`lock_api`** — zero-cost, `no_std` generic wrappers (`Mutex<R, T>`, `RwLock<R, T>`) parameterized over a `RawMutex`/`RawRwLock` trait. It supplies the RAII guards, `MappedMutexGuard`, `ArcMutexGuard`, poisoning-free semantics, and the type-safe API. Anyone can implement a raw lock and get the full ergonomic surface for free.
2. **`parking_lot_core`** — the parking lot itself: a global hash table mapping lock addresses to queues of parked (sleeping) threads, exposing `park`, `unpark_one`, `unpark_all`, `unpark_requeue`, and `unpark_filter`. This is split into its own crate deliberately so its API can evolve without forcing a breaking release of `parking_lot`[^3].
3. **`parking_lot`** — the concrete `RawMutex`/`RawRwLock`/`RawThreadId` implementations wired into `lock_api`, giving the public `Mutex`, `RwLock`, `Condvar`, `Once`, and `ReentrantMutex`.

The fast path for an uncontended lock is a single atomic compare-exchange on the one-byte state word; no call into `parking_lot_core` happens at all. On contention the lock first *spins* a few times (good for very short critical sections), then falls back to parking the thread in the side table. This adaptive spin-then-sleep behavior is what lets one implementation serve both microsecond-scale and long-held locks.

Two policy details matter. `RwLock` uses a **task-fair** algorithm that prevents reader-writer starvation, which std's `RwLock` does not guarantee. And both `Mutex` and `RwLock` implement **eventual fairness**[^4]: on unlock they occasionally hand the lock directly to a waiter rather than letting the releasing thread barge back in, bounding worst-case latency without paying fairness cost on every operation. `RwLock` also supports atomic write→read downgrade and upgradable read locks.

## Production Notes

**std has caught up on many platforms.** Since Rust 1.62 (2022), `std::sync::Mutex`, `RwLock`, and `Condvar` on Linux and several other targets were reimplemented directly on futexes, becoming small and allocation-free[^5]. On those platforms std is now one word and competitive; parking_lot's headline speedups were largely measured against the older, pthread-backed std. Treat the README's benchmark figures as historical unless you re-measure on your target. parking_lot still wins on: sub-byte size (`Mutex` is 1 byte vs std's 1 word), `ReentrantMutex`, upgradable/downgradable `RwLock`, guaranteed-no-spurious-wakeup `Condvar`, and the low-level `parking_lot_core` API.

**No poisoning is a semantics change, not just a convenience.** Code migrated from std loses the "a previous holder panicked" signal. If your invariants depend on detecting a poisoned state after a panic, you must add that yourself.

**Never hold a `parking_lot::Mutex` guard across `.await`.** These are blocking, synchronous locks. Holding one across an await point can deadlock an async executor by parking the OS thread that other tasks need. Use `tokio::sync::Mutex` (or equivalent) when a lock must span suspension points.

**Feature flags interact.** `deadlock_detection` and `send_guard` are mutually incompatible and cannot be enabled together. The deadlock detector is explicitly experimental and off by default. `hardware-lock-elision` (Intel TSX path for `RwLock`) uses inline assembly and requires Rust ≥ 1.59. serde support covers `Mutex`, `RwLock`, and `ReentrantMutex` only — not `Condvar` or `Once`.

**wasm is a sharp edge.** On `wasm32-unknown-unknown`, full support needs nightly with `-C target-feature=+atomics` and a rebuilt std. On stable wasm parking_lot mostly works but will *panic* instead of blocking on a would-be deadlock. Enabling atomics on stable wasm can silently break parking_lot's concurrency guarantees.

**MSRV moves.** The current minimum supported Rust is 1.84, and the maintainer states it "may change at any time" — do not assume a fixed floor if you pin old toolchains.

## When to Use / When Not

**Use when:**
- You need `ReentrantMutex`, upgradable read locks, write→read downgrade, or a spurious-wakeup-free `Condvar` that std doesn't provide.
- You want to build your own synchronization primitive on top of `parking_lot_core`'s park/unpark table.
- You target a platform where std still uses the heavier pthread-backed locks, or you want the smallest possible lock footprint for fine-grained locking.
- You prefer poisoning-free ergonomics (`lock()` returning the guard directly).

**Avoid when:**
- You're on a modern std/target and want zero extra dependencies — post-1.62 std is often good enough.
- You need lock poisoning to guard against panic-corrupted state.
- Your lock must be held across `async` await points — use an async-aware mutex instead.
- You want a `no_std` bare-metal spinlock with no thread parking — parking assumes an OS thread to suspend.

## Alternatives

- rust-lang/rust (`std::sync`) — since 1.62 its futex-based locks are small and fast on major targets; use when you want no dependencies plus poisoning semantics.
- tokio-rs/tokio (`tokio::sync::Mutex`) — use when a lock must be held across `.await`; parking_lot is synchronous and will block the executor thread.
- crossbeam-rs/crossbeam — use when you need channels, epoch-based reclamation, or other lock-free structures rather than blocking mutexes.
- mvdnes/spin-rs — use in `no_std`/bare-metal contexts where there is no OS to park threads on and a raw spinlock is acceptable.
- kprotty/usync — use when you want a parking_lot-style API with a different internal locking strategy and word-sized locks.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-05 | First release; WebKit ParkingLot design ported to Rust[^1]. |
| 0.9 | 2019 | Split of `lock_api` / `parking_lot_core` stabilized as the layered design. |
| 0.11 | 2020 | Widely adopted release across the ecosystem. |
| 0.12 | 2022 | Current major line; dependency and `lock_api` updates. |

Exact patch dates within the 0.12 line are published on crates.io; the table above lists only milestones the maintainer's changelog corroborates at year granularity.

## References

[^1]: Filip Pizlo, "Locking in WebKit" — the `WTF::ParkingLot` design parking_lot is based on. https://webkit.org/blog/6161/locking-in-webkit/
[^2]: parking_lot README, benchmark summary (x86_64 Linux). https://github.com/Amanieu/parking_lot
[^3]: `parking_lot_core` crate documentation. https://docs.rs/parking_lot_core/
[^4]: "Eventual fairness" — WebKit changeset referenced by the README. https://trac.webkit.org/changeset/203350
[^5]: Rust 1.62 release notes — std `Mutex`/`RwLock`/`Condvar` reimplemented on futex. https://blog.rust-lang.org/2022/06/30/Rust-1.62.0.html

## Tags

rust, concurrency, synchronization, mutex, rwlock, locking, systems-programming, no-std, threads, primitives
