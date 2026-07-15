# libevent/libevent

> A C event-notification library that wraps the platform's fastest readiness poller (epoll, kqueue, IOCP-ish) behind one callback-driven event loop.

[GitHub repo](https://github.com/libevent/libevent) ·
[Official website](https://libevent.org) ·
[License: BSD-3-Clause](https://github.com/libevent/libevent/blob/master/LICENSE)

## Overview

libevent is a portable C library for reactor-style asynchronous I/O. You register file descriptors, timers, and signals with an `event_base`, and it dispatches callbacks when those become ready, using the best mechanism the host offers (`epoll` on Linux, `kqueue` on BSD/macOS, `/dev/poll` and event ports on Solaris/illumos, `poll`/`select` as fallbacks, and `select`-based support on Windows)[^1]. It originated with Niels Provos in the early 2000s and has been maintained primarily by Nick Mathewson since; it is one of the oldest and most widely embedded async I/O libraries in the C ecosystem[^2].

Its reach is largely historical and infrastructural rather than fashionable: Chromium, Tor, Memcached, Transmission, and tmux have all shipped against it, and it remains a common dependency in older network daemons. The defining tension of the project today is age versus alternatives. The core reactor is mature and battle-tested, but the library also grew a stack of higher-level helpers (a buffered-I/O layer, async DNS, a minimal HTTP server, an RPC layer) whose quality and upkeep vary, and releases have become infrequent — the 2.1.x line is years old and 2.2 has sat in alpha for a long time[^3].

libevent predates and coexists with `libev` (a leaner reimplementation of the same idea) and `libuv` (Node.js's loop, which does real Windows IOCP). Choosing libevent in 2026 usually means you value its ubiquity, its evbuffer/bufferevent abstractions, or an existing codebase — not raw performance leadership.

## Getting Started

```sh
# Debian/Ubuntu
sudo apt install libevent-dev
# macOS
brew install libevent
# vcpkg
vcpkg install libevent
# from source (autoconf is deprecated since 2.2; prefer CMake)
mkdir build && cd build && cmake .. && make && sudo make install
```

```c
// tick.c — minimal event_base with a one-shot timer. Build: cc tick.c -levent
#include <event2/event.h>
#include <stdio.h>

static void on_timeout(evutil_socket_t fd, short events, void *arg) {
    struct event_base *base = arg;
    printf("tick\n");
    event_base_loopbreak(base);          // exit the loop after one tick
}

int main(void) {
    struct event_base *base = event_base_new();
    struct timeval one_sec = { 1, 0 };
    struct event *ev = event_new(base, -1, EV_TIMEOUT, on_timeout, base);
    event_add(ev, &one_sec);
    event_base_dispatch(base);           // blocks until no active events
    event_free(ev);
    event_base_free(base);
    return 0;
}
```

Use the `event2/` headers (explicit `event_base` passed everywhere), not the legacy `event.h` global-base API (`event_init`, `event_dispatch`), which relies on a process-wide "current" base and is effectively deprecated[^4].

## Architecture / How It Works

The core is a **reactor**: `event_base` owns a backend chosen at runtime, a set of registered `struct event`s, and a min-heap of pending timeouts. Each `event` binds a file descriptor (or signal) to a mask (`EV_READ`, `EV_WRITE`, `EV_TIMEOUT`, `EV_SIGNAL`, optionally `EV_PERSIST` for auto-rearm and `EV_ET` for edge-triggered). `event_base_dispatch()` runs the loop: poll the backend, fire ready callbacks in priority order, expire timers. Backends are selected by a priority list but can be forced via `EVENT_NOKQUEUE`-style environment flags for debugging[^1].

Above the raw event layer sit several optional subsystems, and this is where the coupling story matters:

- **evbuffer** — a growable I/O buffer built from a linked chain of segments, with `evbuffer_add_reference` (zero-copy), `evbuffer_add_file` (sendfile/mmap), and search helpers. It is the substrate for everything buffered.
- **bufferevent** — a stream abstraction over a socket (or a filter, or a pair) with read/write callbacks, watermarks, and token-bucket rate limiting. `bufferevent_openssl_*` layers TLS via OpenSSL, and an mbedTLS variant exists. This is the layer most application code should target.
- **evdns** — an asynchronous DNS resolver written from scratch (not a `getaddrinfo` wrapper), used to avoid blocking the loop on name resolution.
- **evhttp** — a minimal HTTP/1.1 client and server, intended for control endpoints and simple embedded servers.
- **evrpc** — a lightweight RPC layer, largely legacy.

Threading is opt-in and easy to get wrong. An `event_base` is **not thread-safe by default**; you must call `evthread_use_pthreads()` / `evthread_use_windows_threads()` before creating bases if you intend to touch a base from multiple threads. The idiomatic pattern is one `event_base` per thread. Cross-thread wakeups go through `event_base_loopbreak`/`loopexit` (safe) or `bufferevent`s created with `BEV_OPT_THREADSAFE`[^5].

## Production Notes

- **evhttp is not a production web server.** It has no HTTP/2, limited header/parser hardening, and a history of parsing bugs. It is fine for a status endpoint embedded in a daemon; it is the wrong tool for a public-facing HTTP service.
- **evdns has had multiple CVEs** (notably a cluster of DNS-parsing issues disclosed in 2016)[^6]. If you resolve untrusted names, keep the library patched and consider whether you need the built-in resolver at all.
- **fork() requires `event_reinit(base)`** in the child; kernel poller state (epoll/kqueue fds) does not survive a fork intact, and skipping the reinit produces silent, hard-to-debug misfires.
- **Windows scaling is limited.** The core loop uses `select` on Windows, capped by `FD_SETSIZE`-style limits; true IOCP is only reached through `bufferevent_async`, not the general event path. Treat Windows as "supported" rather than "high-scale."
- **Signals are per-base and racy across bases.** Register signal events on exactly one base, and be aware that signal delivery interacts with the internal socketpair used to wake the loop.
- **Many-timer workloads** benefit from `event_base_init_common_timeout`, which buckets equal timeouts to avoid heap churn — without it, tens of thousands of identical timeouts degrade.
- **Version coexistence:** 2.0 and 2.1 install side by side via versioned symbols/headers, but mixing `event.h` (legacy) and `event2/` headers in one program is a common source of subtle ABI/global-base confusion.
- **Maintenance cadence is slow.** Interpret the recent commit activity carefully: the tree gets fixes, but tagged stable releases are rare, so distributions may ship a years-old point release. Pin and test the exact version you deploy.

## When to Use / When Not

**Use when:**
- You need a portable C reactor and want the most widely deployed, most-linked option.
- You want evbuffer/bufferevent's buffered-I/O and TLS-filter abstractions rather than hand-rolling readiness handling.
- You're maintaining or extending software that already depends on it.

**Avoid when:**
- You target Windows at scale and need real IOCP — use libuv.
- You want the leanest, fastest possible core loop with no HTTP/DNS/RPC baggage — use libev.
- You're writing C++ and want coroutines and a modern proactor model — use Asio.
- You're Linux-only and chasing maximum throughput — an `io_uring` design will beat readiness polling.

## Alternatives

- libuv/libuv — use when you need first-class Windows IOCP and a Node-style handle/request model, not just a POSIX-shaped readiness loop.
- libev (schmorp.de, no canonical GitHub repo) — use when you want a smaller, faster core loop and none of libevent's HTTP/DNS/RPC helpers.
- chriskohlhoff/asio — use when you're in C++ and want a proactor with coroutine support instead of C callbacks.
- axboe/liburing — use when you're Linux-only and want io_uring throughput rather than portable readiness polling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2000 | Initial release by Niels Provos; select/poll/kqueue reactor[^2]. |
| 1.4.x | 2008–2012 | Long-lived stable line; evhttp/evdns maturation. |
| 2.0.x | 2010–2012 | Major redesign: explicit `event_base` API (`event2/`), bufferevents, opt-in threading, OpenSSL bufferevents, IPv6-aware evdns[^7]. |
| 2.1.x | 2013– | Current stable line; 2.1.12-stable is a widely packaged point release[^3]. |
| 2.2 | (alpha) | In development; autoconf deprecated in favor of CMake[^4]. |

## References

[^1]: libevent Reference Manual — event_base backends and the event loop. https://libevent.org/doc/
[^2]: Niels Provos and Nick Mathewson, "libevent" project history. https://libevent.org/
[^3]: libevent releases (GitHub). https://github.com/libevent/libevent/releases
[^4]: libevent README / Documentation/Building — "since 2.2 autoconf is deprecated," CMake is preferred. https://github.com/libevent/libevent/blob/master/README.md
[^5]: Nick Mathewson, "Fast portable non-blocking network programming with Libevent" (libevent book), threading chapter. https://libevent.org/libevent-book/
[^6]: libevent security advisories (evdns DNS-parsing CVEs, 2016). https://github.com/libevent/libevent/security/advisories
[^7]: libevent 2.0 changelog / "What's new in Libevent 2.0." https://libevent.org/libevent-book/

## Tags

c, event-loop, async-io, networking, reactor, epoll, kqueue, cross-platform, libevent, non-blocking, sockets
