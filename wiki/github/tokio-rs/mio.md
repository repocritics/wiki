# tokio-rs/mio

> Metal I/O — a thin, non-blocking readiness layer over epoll/kqueue/IOCP for Rust, and the reactor underneath Tokio.

[GitHub repo](https://github.com/tokio-rs/mio) ·
[API docs](https://docs.rs/mio) ·
[License: MIT](https://github.com/tokio-rs/mio/blob/master/LICENSE)

## Overview

Mio ("Metal I/O") is a low-level library that wraps the operating system's
event-notification primitives — `epoll` on Linux, `kqueue` on the BSDs and
macOS, and a `wepoll`/AFD strategy on Windows — behind a single portable API[^1].
It provides non-blocking TCP, UDP, and Unix domain sockets plus a `Poll` type
that blocks until one of the registered sources is ready, and nothing else. It
is deliberately small: file I/O, timers, thread pools, and any notion of a task
scheduler are explicit non-goals, left to higher layers[^2].

The library exists mostly so that other libraries don't have to. Its primary
consumer is Tokio, whose reactor is built directly on `mio::Poll`; if you are
writing application code you almost certainly want Tokio or `async-std`-style
runtimes instead, and the README says as much[^1]. Mio's audience is the small
set of people writing event loops, custom runtimes, or protocol libraries that
need the OS readiness signal without an async runtime's opinions attached.

The defining design choice is that Mio is a *readiness* interface, not a
*completion* one. You register interest in a socket, poll tells you it is
"probably readable," and you then attempt the syscall yourself and handle
`WouldBlock`. This maps cleanly onto epoll and kqueue but fits Windows IOCP
(a completion model) awkwardly — a mismatch that shaped much of Mio's history
and its eventual move to the AFD-based `wepoll` approach on Windows[^3].

## Getting Started

```toml
[dependencies]
mio = { version = "1", features = ["os-poll", "net"] }
```

The `os-poll` and `net` features are not on by default; without them the socket
types and `Poll` are not compiled in.

```rust
use mio::net::TcpListener;
use mio::{Events, Interest, Poll, Token};

const SERVER: Token = Token(0);

fn main() -> std::io::Result<()> {
    let mut poll = Poll::new()?;
    let mut events = Events::with_capacity(128);

    let addr = "127.0.0.1:13265".parse().unwrap();
    let mut server = TcpListener::bind(addr)?;
    poll.registry()
        .register(&mut server, SERVER, Interest::READABLE)?;

    loop {
        poll.poll(&mut events, None)?; // blocks until a source is ready
        for event in events.iter() {
            if event.token() == SERVER {
                // Drain until WouldBlock — events are edge-triggered.
                loop {
                    match server.accept() {
                        Ok((_conn, _addr)) => { /* register _conn ... */ }
                        Err(ref e) if e.kind() == std::io::ErrorKind::WouldBlock => break,
                        Err(e) => return Err(e),
                    }
                }
            }
        }
    }
}
```

## Architecture / How It Works

Four types carry the whole model. `Poll` owns the OS selector. `Registry` is a
cloneable handle to that selector, so a source can be registered from another
thread than the one calling `poll()`. `Token` is a `usize` you associate with a
source at registration and get back on each event — Mio stores no map for you,
so the token is how you route an event to your own connection state. `Events` is
a reusable buffer you hand to `poll()` and iterate afterward.

Registration takes an `Interest` (`READABLE`, `WRITABLE`, or their union). Since
the 0.7 rewrite, Mio's readiness is **edge-triggered**: `poll()` reports a
transition to ready once, and you are expected to keep issuing the syscall until
it returns `WouldBlock`. Forgetting to drain is the single most common Mio bug —
the event never fires again and the connection appears to hang.

Platform backends live behind one `sys` module. Linux uses `epoll` (with
`eventfd` for the cross-thread `Waker`); the BSDs and macOS use `kqueue`. Windows
does not expose an epoll-style readiness API, so Mio uses the `wepoll` strategy:
it drives sockets through the undocumented AFD (`\Device\Afd`) interface to
synthesize readiness events on top of IOCP[^3]. This is why the Windows story was
historically the least stable part of the codebase and why several `mio_unsupported_*`
cfg flags exist to force fallback `poll(2)`/`pipe(2)` implementations when the
"best" backend misbehaves on a given target[^1].

`Waker` is the one piece of cross-thread machinery Mio provides: a handle that,
when woken from any thread, causes a blocked `poll()` to return. Runtimes use it
to inject work into an otherwise sleeping event loop. Everything above this — the
executor, futures, timers, connection pools — is Tokio's job, not Mio's.

## Production Notes

- **You probably shouldn't use Mio directly.** In practice "using Mio" means
  using Tokio, which vendors and version-pins it. Reaching for raw Mio is a
  signal you are writing a runtime; if you are not, the readiness/`WouldBlock`
  bookkeeping is error-prone surface you don't need.
- **Edge-triggered draining is mandatory.** Every readable/writable event must be
  serviced in a loop until `WouldBlock`. Code ported from level-triggered epoll
  assumptions will stall silently.
- **Feature flags are opt-in.** A dependency on `mio = "1"` with default features
  compiles almost nothing usable; `os-poll` and `net` are required for the socket
  APIs. This trips up first-time users whose code fails to resolve `mio::net`.
- **MSRV moves with minor versions.** Mio fixes its Minimum Supported Rust
  Version within a `1.x` line but may raise it on a minor bump: v0.8 needed Rust
  1.46, v1.0 needs 1.70, and v1.1/v1.2 need 1.71[^1]. Pin a minor version if you
  support old toolchains.
- **The 0.6 → 0.7 jump was a rewrite, not an upgrade.** `Ready`/`PollOpt` were
  replaced by `Interest`, `Registry` was introduced, and default triggering
  changed. Nearly every non-trivial consumer needed code changes; treat any
  migration across that boundary as a port[^4].
- **Windows behavior is a class of its own.** Because readiness is emulated over
  AFD, edge cases (half-closed sockets, certain error propagations) have
  historically differed from Unix. Test the Windows target explicitly rather than
  assuming Unix parity.
- **Maturity, not abandonment.** With ~24 open issues against 7k stars and
  commits landing through mid-2026, Mio is in steady maintenance: small surface,
  slow cadence, few open threads. Low churn here is the intended state, not a
  stalled project.

## When to Use / When Not

**Use when:**
- You are building an async runtime, custom event loop, or protocol library and
  need portable OS readiness without an executor's opinions.
- You want zero runtime allocation and a direct, auditable line to `epoll`/`kqueue`.
- You need cross-platform non-blocking sockets and are willing to own the
  `WouldBlock` state machine yourself.

**Avoid when:**
- You are writing an application or service — use Tokio (or another runtime)
  built on Mio instead.
- You need timers, file I/O, DNS, TLS, or a task scheduler; Mio provides none of
  these by design.
- You want completion-based I/O (io_uring, Windows IOCP semantics) as a
  first-class model — Mio's readiness abstraction is a different shape. See
  tokio-rs/tokio-uring for that direction.

## Alternatives

- tokio-rs/tokio — the runtime you almost certainly want; Mio is its reactor. Use instead when you need async application code, not a reactor.
- smol-rs/polling — a smaller epoll/kqueue/IOCP wrapper powering the `smol`/async-std ecosystem. Use when you want a lighter, more focused selector abstraction.
- bytecodealliance/rustix — safe syscall bindings including epoll/kqueue. Use when you want direct OS access and are willing to build your own event loop.
- tokio-rs/tokio-uring — completion-based I/O on Linux `io_uring`. Use when you specifically need io_uring semantics rather than readiness polling.
- libuv/libuv — the C event loop behind Node.js. Use when you need a battle-tested cross-language loop outside the Rust ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.6.21 | 2019-11-27 | Last of the long-lived 0.6 line; readiness API with `Ready`/`PollOpt`[^4]. |
| 0.7.0 | 2020-03-02 | Ground-up rewrite: `Registry`, `Interest`, edge-triggered default, Windows moved toward wepoll/AFD[^4]. |
| 0.8.0 | 2021-11-13 | API cleanups atop the 0.7 model; MSRV Rust 1.46. |
| 1.0.0 | 2024-06-14 | First stable major; MSRV Rust 1.70, long-term API commitment[^5]. |
| 1.1.0 | 2025-10-17 | Minor additions; MSRV raised to Rust 1.71. |
| 1.2.0 | 2026-03-27 | Continued 1.x maintenance line. |

## References

[^1]: Mio README — features, platform list, wepoll note, MSRV policy, and unsupported cfg flags. https://github.com/tokio-rs/mio/blob/master/README.md
[^2]: Mio README, "Non-goals" — file operations, thread pools, and timers are explicitly out of scope. https://github.com/tokio-rs/mio/blob/master/README.md
[^3]: wepoll — Windows AFD-based epoll emulation used by Mio's Windows backend. https://github.com/piscisaureus/wepoll
[^4]: `mio::Poll` API documentation — readiness model, edge vs level triggering, and registration semantics. https://docs.rs/mio/latest/mio/struct.Poll.html
[^5]: Mio CHANGELOG — release notes across the 0.7/0.8/1.0 lines. https://github.com/tokio-rs/mio/blob/master/CHANGELOG.md

## Tags

rust, networking, non-blocking, event-loop, epoll, kqueue, iocp, async-io, low-level, reactor, cross-platform
