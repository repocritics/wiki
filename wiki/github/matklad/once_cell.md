# matklad/once_cell

> Single-assignment cells and macro-free lazy statics for Rust — the crate so
> successful its API was absorbed into the standard library.

[GitHub repo](https://github.com/matklad/once_cell) ·
[docs.rs](https://docs.rs/once_cell) ·
[License: Apache-2.0](https://github.com/matklad/once_cell/blob/master/LICENSE-APACHE)

## Overview

`once_cell` is a small Rust library by Aleksey Kladov (matklad, also the
original author of rust-analyzer and IntelliJ Rust), first published in
2018[^1]. It provides `OnceCell<T>` — a cell that can be written to at most
once and read as a plain `&T` afterwards — and `Lazy<T>`, a value initialized
on first access. Its practical achievement was replacing the `lazy_static!`
macro with ordinary types: no macro syntax, no hidden statics, direct `&T`
access instead of guard types.

The crate's star count (~2,100) wildly understates its footprint. It is one of
the most-downloaded crates on crates.io and sits in the dependency tree of a
large fraction of the Rust ecosystem, including rustc and cargo themselves.
The reason stars are low is that nobody browses it — it is plumbing.

The defining fact of its current life: the API was upstreamed into `std`.
`OnceCell` / `OnceLock` stabilized in Rust 1.70 (June 2023)[^2][^3], and
`LazyCell` / `LazyLock` in Rust 1.80 (July 2024)[^4]. The crate is therefore
in deliberate maintenance mode — feature-complete, still patched (last push
March 2026), but for new code on a modern toolchain it is largely superseded
by the standard library, with a few exceptions covered below.

## Getting Started

```bash
cargo add once_cell
```

```rust
use std::{collections::HashMap, sync::Mutex};
use once_cell::sync::Lazy;

static GLOBAL_DATA: Lazy<Mutex<HashMap<i32, String>>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert(13, "Spica".to_string());
    Mutex::new(m)
});

fn main() {
    println!("{:?}", GLOBAL_DATA.lock().unwrap());
}
```

On Rust ≥ 1.80, the equivalent without any dependency is
`std::sync::LazyLock` with the same shape.

## Architecture / How It Works

The crate is organized along one axis — synchronization strategy — with three
modules exposing the same conceptual API[^5]:

- **`unsync`** — single-threaded. `OnceCell` is essentially a
  `Cell<Option<T>>`; not `Sync`, zero synchronization overhead. The compiler
  enforces thread confinement.
- **`sync`** — thread-safe and *blocking*. If two threads race to initialize,
  one runs the closure and the other parks until it finishes, so the init
  closure runs exactly once. The default implementation uses an atomic state
  word plus a waiter queue (the same design as `std::sync::Once`); the
  `parking_lot` feature swaps the blocking primitive for `parking_lot_core`.
- **`race`** — thread-safe and *lock-free*. Competing threads may all run the
  initialization; the first `compare_exchange` wins and the losers' results
  are discarded (`OnceBox` frees the losing allocation). No blocking, which
  makes it usable where parking a thread is not (and it is the piece of the
  crate that never made it into `std`).

The core trick that made the design worth stabilizing: because a cell is
written at most once, `get` can hand out a plain `&T` for the rest of the
program's life — no `MutexGuard`, no `Ref`, no runtime borrow bookkeeping.
`Lazy<T, F>` is just an `OnceCell<T>` paired with the init closure, with
`Deref` triggering `get_or_init`.

Error behavior differs from `lazy_static`: if the init closure panics, the
panic propagates and the cell stays *empty* — a later access retries
initialization rather than hitting a poisoned state.

## Production Notes

- **On modern Rust, prefer `std`.** `OnceLock`/`LazyLock` cover the common
  cases since 1.70/1.80. Keeping `once_cell` is harmless (it compiles in
  well under a second) but it is one more supply-chain entry for no gain.
  Migration is mostly mechanical renaming.
- **The `race` module is the surviving niche.** `std` has no lock-free,
  non-blocking once-init. If you need first-writer-wins semantics in signal
  handlers, allocators, or other no-park contexts, `once_cell::race` (or
  `OnceBox`) is still the tool.
- **`no_std` support** is another reason the crate persists: `race` works
  with `alloc` alone, and the `critical-section` feature enables `sync`-like
  behavior on bare-metal targets where `std`'s primitives are unavailable.
- **Reentrancy is a footgun.** Calling `get_or_init` from inside its own init
  closure deadlocks (`sync`) or errors (`unsync`) — the docs warn about it,
  but it typically shows up as a hang, not a message, when two lazy statics
  accidentally initialize each other.
- **Statics never drop.** A `static Lazy<T>` runs no destructor at process
  exit. File handles or sockets stored in one are released by the OS, not by
  `Drop` — relevant if your type buffers writes.
- **Maintenance mode is a feature here, not a risk.** The API has been stable
  through the entire 1.x line since 2019; changes are rare and conservative.
  This is finished software with an exit path (`std`) if it ever stopped.

## When to Use / When Not

**Use when:**
- Your MSRV is below 1.70 (or below 1.80 and you need lazy statics).
- You need lock-free once-init (`race` module) — signal handlers, allocators.
- You target `no_std` with `alloc` or `critical-section`.
- It is already in your tree via dependencies (it is; leave it alone).

**Avoid when:**
- You are on Rust ≥ 1.80 writing new code — `std::sync::OnceLock` and
  `LazyLock` do the same job with zero dependencies.
- You need async-aware lazy initialization — `OnceCell::get_or_init` blocks
  the thread; in async code use an async-native cell instead.

## Alternatives

- std `OnceLock` / `LazyLock` — the default on Rust ≥ 1.80; use unless you
  need `race`, `no_std`, or an older MSRV.
- rust-lang-nursery/lazy-static.rs — the macro-based predecessor; only for
  legacy codebases, itself long in maintenance mode.
- indiv0/lazycell — older single-thread lazy cell; largely superseded by
  both `once_cell` and `std`.
- niklasf/double-checked-cell — double-checked-locking variant the README
  lists as related; use only if its specific semantics fit.
- tokio-rs/tokio — `tokio::sync::OnceCell` when initialization itself must
  `await` instead of blocking a thread.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018 | Initial release; macro-free alternative to `lazy_static`[^1]. |
| 1.0 | 2019 | API stabilized; the 1.x line has been unbroken since. |
| — | 2023-06 | Rust 1.70 stabilizes `OnceCell`/`OnceLock` in `std`[^2][^3]. |
| — | 2024-07 | Rust 1.80 stabilizes `LazyCell`/`LazyLock` in `std`[^4]. |
| 1.x | 2024–2026 | Maintenance releases; last push 2026-03. |

## References

[^1]: once_cell README. https://github.com/matklad/once_cell#overview
[^2]: rust-lang/rust PR #105587, "Move `OnceCell` and `OnceLock` out of `sync`" (std adoption path). https://github.com/rust-lang/rust/pull/105587
[^3]: Rust Team, "Announcing Rust 1.70.0" — 2023-06-01. https://blog.rust-lang.org/2023/06/01/Rust-1.70.0.html
[^4]: Rust Team, "Announcing Rust 1.80.0" — 2024-07-25. https://blog.rust-lang.org/2024/07/25/Rust-1.80.0.html
[^5]: once_cell API documentation. https://docs.rs/once_cell/

## Tags

rust, lazy-evaluation, lazy-static, once-cell, concurrency, library, no-std, memoization, synchronization, standard-library-precursor
