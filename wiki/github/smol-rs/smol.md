# smol-rs/smol

> A small, composable async runtime for Rust that is mostly a thin facade over a family of independently useful crates.

[GitHub repo](https://github.com/smol-rs/smol) ·
[docs.rs](https://docs.rs/smol) ·
[License: Apache-2.0 OR MIT](https://github.com/smol-rs/smol/blob/master/LICENSE-APACHE)

## Overview

`smol` is an async runtime for Rust originally written by Stjepan Glavina (also
author of `async-std` and much of `crossbeam`) and first published in early
2020[^1]. Its distinguishing design choice is stated plainly in the README: the
`smol` crate "simply re-exports other smaller async crates." The real substance
lives in a dozen independently versioned crates under the `smol-rs` org —
`async-executor`, `async-io`, `polling`, `async-channel`, `blocking`,
`futures-lite`, and others — that `smol` bundles into one ergonomic surface.

This modularity is the project's defining tension. The component crates are so
generally useful that they are depended upon far beyond `smol` itself:
`polling`, `async-channel`, `event-listener`, and `concurrent-queue` show up
transitively in a large share of the async Rust ecosystem, including projects
that otherwise run on Tokio. So `smol`'s footprint is much larger than its star
count suggests — but the `smol` facade crate is a minority runtime choice. Most
libraries (axum, tonic, reqwest, sqlx defaults) assume Tokio, and using them
under `smol` requires the `async-compat` shim, which starts a Tokio runtime
alongside `smol`'s reactor[^2].

The project is community-maintained under the `smol-rs` organization after
Glavina stepped back, with John Nunley (notgull) the most active core-crate
maintainer in recent years. The stated goal is a small, auditable,
fast-compiling async stack rather than a maximal-throughput one.

## Getting Started

```bash
cargo add smol
```

```rust,no_run
use smol::{io, net, prelude::*, Unblock};

fn main() -> io::Result<()> {
    smol::block_on(async {
        let mut stream = net::TcpStream::connect("example.com:80").await?;
        let req = b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n";
        stream.write_all(req).await?;

        // Unblock offloads the blocking stdout handle onto a thread pool.
        let mut stdout = Unblock::new(std::io::stdout());
        io::copy(stream, &mut stdout).await?;
        Ok(())
    })
}
```

`smol::block_on` drives a future to completion on the current thread;
`smol::spawn` schedules onto a lazily-started global multi-threaded executor.
For a proc-macro-free `async fn main` or an explicit executor, use the
companion `smol-macros` crate rather than reaching for `#[tokio::main]`-style
attributes.

## Architecture / How It Works

`smol` is a facade; understanding it means understanding the layers underneath:

- **`polling`** — a portable wrapper over the OS readiness APIs (epoll on Linux,
  kqueue on the BSDs/macOS, event ports on illumos/Solaris, IOCP/wepoll on
  Windows). This is the bottom of the I/O stack.
- **`async-io`** — a single reactor thread built on `polling` that registers
  file descriptors and timers, plus the `Async<T>` adapter that turns any
  `AsRawFd`/`AsRawSocket` type into a future-aware one. Timers live here too.
- **`async-executor`** — the task scheduler. `Executor` is a multi-threaded,
  work-stealing-ish executor; `LocalExecutor` is single-threaded. `smol::spawn`
  targets a lazily-initialized global `Executor`.
- **`blocking`** — a dynamically-sized thread pool for work that has no async
  equivalent. `Unblock<T>` and the `unblock()` function move blocking I/O onto
  it.
- **`async-fs` / `async-net` / `async-process`** — filesystem, socket, and
  subprocess wrappers layered on the above; **`async-channel` / `async-lock` /
  `event-listener`** provide channels, locks, and the notification primitive
  they are built on; **`futures-lite`** is a lighter reimplementation of the
  common `futures` surface that keeps dependency and compile costs down.

A key architectural consequence: `async-fs` is **not** kernel-async I/O — it is
ordinary blocking `std::fs` calls dispatched onto the `blocking` thread pool
(io_uring-style async disk I/O is not part of this stack). Networking and
timers, by contrast, are true readiness-based async through the `polling`
reactor. Reactor and executor are decoupled, which is what lets you mix
`block_on`, a global `spawn`, and hand-built `Executor` instances in one
program.

## Production Notes

- **Ecosystem lock-in is the main cost.** Most async libraries are written
  against Tokio's traits and runtime handle. Running them under `smol` means
  wrapping their futures/streams in `async-compat`, which spawns a Tokio runtime
  next to `smol`'s reactor — two runtimes in one process, with the attendant
  overhead and reasoning burden[^2]. If your dependency tree is Tokio-heavy,
  `smol` fights you.
- **"Async" filesystem is thread-pool offload.** Heavy `async-fs` workloads are
  bounded by the `blocking` pool's thread count and by page-cache/syscall
  latency, not by any kernel async path. Do not expect io_uring-class disk
  throughput.
- **Single global reactor thread.** All `async-io`-registered I/O and timers are
  serviced by one reactor thread. It is efficient for typical workloads but is a
  different scaling shape from Tokio's per-worker I/O drivers.
- **No macro runtime by default.** There is no `#[smol::main]`. This is
  deliberate (fast compiles, no proc-macro dependency), but it surprises people
  migrating from Tokio; use `smol-macros` or write `fn main` + `block_on`.
- **You are already using parts of it.** Tokio-based builds commonly pull in
  `polling`, `async-channel`, or `event-listener` transitively, so `smol-rs`
  crates in your `Cargo.lock` do not mean someone chose the `smol` runtime.
- **Conservative MSRV.** 1.85, informally pinned to the Rust in Debian Stable
  and advanced only for major ecosystem shifts or security fixes.

## When to Use / When Not

**Use when:**
- You want a small, auditable async stack with fast compile times and a shallow
  dependency tree.
- You are embedding async into a mostly-synchronous program and want
  `block_on` plus occasional `spawn` without adopting a large runtime.
- You want to compose executors and reactors explicitly rather than accept a
  monolithic runtime's defaults.

**Avoid when:**
- Your stack leans on Tokio-native libraries (axum, tonic, tower, most gRPC/DB
  clients) — the `async-compat` tax is not worth it.
- You need io_uring-class disk or completion-based network I/O.
- You want the largest ecosystem of runtime-specific middleware, tracing
  integrations, and hiring familiarity; that is Tokio.

## Alternatives

- tokio-rs/tokio — the default choice; use it when you need the ecosystem, the
  tooling, or io-driver throughput at scale.
- async-rs/async-std — a std-shaped API built on the same `smol-rs` crates, but
  deprecated in 2025; new projects should not start here.
- compio-rs/compio — use instead when you specifically want completion-based
  (io_uring / IOCP) async I/O.
- DataDog/glommio — use for thread-per-core, io_uring, high-throughput Linux
  network services.
- embassy-rs/embassy — use for `no_std` embedded async on microcontrollers,
  where `smol`'s OS-reactor model does not apply.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-02-04 | `smol-rs/smol` first pushed[^1]. |
| 0.1 | 2020 | Initial releases; single-crate experiment by Stjepan Glavina. |
| 1.0 | 2020 | Stabilized facade re-exporting the component crates. |
| 2.0 | 2023 | Breaking release aligning with newer `async-io`/component majors[^3]. |
| — | 2026-06 | Actively maintained under `smol-rs`; last push 2026-06-27, MSRV 1.85. |

## References

[^1]: `smol-rs/smol` repository metadata (created 2020-02-04, 5,011 stars,
Apache-2.0 OR MIT). https://github.com/smol-rs/smol
[^2]: `async-compat` — adapter for running Tokio-based futures and I/O under
other runtimes. https://docs.rs/async-compat
[^3]: `smol` release history on crates.io. https://crates.io/crates/smol/versions

## Tags

rust, async, async-runtime, futures, executor, networking, concurrency, io, event-loop, minimalist
