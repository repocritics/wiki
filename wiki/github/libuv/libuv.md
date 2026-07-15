# libuv/libuv

> The cross-platform C library that gives Node.js its event loop — one async I/O abstraction over epoll, kqueue, IOCP, and event ports.

[GitHub repo](https://github.com/libuv/libuv) ·
[Official website](https://libuv.org/) ·
[License: MIT](https://github.com/libuv/libuv/blob/v1.x/LICENSE)

## Overview

libuv is a C library providing a single-threaded, event-loop-based asynchronous
I/O API that behaves the same on Linux, macOS, the BSDs, Windows, and several
Unix variants[^1]. It was carved out of Node.js around 2011 to give Node one
abstraction over the wildly different async primitives each OS exposes:
`epoll` on Linux, `kqueue` on macOS/BSD, IOCP (completion ports) on Windows,
and event ports on Solaris/illumos. Early versions wrapped libev and libeio on
Unix; that dependency was removed before 1.0 and the loop is now implemented
natively per platform[^2].

Node.js is still the primary consumer and the main driver of the roadmap, but
libuv is a standalone library used well outside it: Julia, Luvit, Neovim (via
the `luv` binding), uvloop (the fast asyncio event loop for Python), and pyuv
all embed it[^1]. If you are writing a cross-platform network daemon or a
language runtime in C/C++ and need Windows/Unix parity for sockets, timers,
child processes, and filesystem I/O, libuv is the default answer.

The defining tension is that "asynchronous I/O" means two different things
inside libuv. Network I/O is genuinely event-driven through the OS readiness/
completion APIs. Filesystem I/O and DNS resolution are *not* — on most platforms
they are run on a fixed-size worker thread pool because no portable async file
API exists. Understanding which operations are real async and which are
thread-pool-backed is the single most important thing to know before shipping
libuv in production.

## Getting Started

libuv is a C dependency; install it from your package manager or build from
source (autotools or CMake; CMake is the only supported path on Windows)[^1].

```bash
# Debian/Ubuntu
sudo apt install libuv1-dev
# macOS
brew install libuv
# vcpkg (cross-platform)
vcpkg install libuv
```

```c
// tick.c — a one-shot timer on the default loop
#include <uv.h>
#include <stdio.h>

static void on_timer(uv_timer_t* handle) {
    printf("tick\n");
    uv_stop(handle->loop);          // ask the loop to unwind
}

int main(void) {
    uv_loop_t* loop = uv_default_loop();
    uv_timer_t timer;
    uv_timer_init(loop, &timer);
    uv_timer_start(&timer, on_timer, 1000, 0);   // fire once after 1000 ms
    uv_run(loop, UV_RUN_DEFAULT);                // blocks until no active handles
    return uv_loop_close(loop);
}
```

```bash
cc tick.c -luv -o tick && ./tick
```

## Architecture / How It Works

The two central concepts are **handles** and **requests**. Handles are
long-lived objects registered with a loop (`uv_tcp_t`, `uv_udp_t`,
`uv_timer_t`, `uv_process_t`, `uv_async_t`); they stay active until explicitly
closed. Requests are short-lived operations against a handle or the loop
(`uv_write_t`, `uv_fs_t`, `uv_getaddrinfo_t`, `uv_connect_t`). Handles keep the
loop alive; when the last active handle closes, `uv_run` returns.

Each `uv_run` iteration walks a fixed sequence of phases: update the loop's
cached time, run due timers, run pending callbacks, run idle/prepare handles,
compute a poll timeout, block in the OS poll (`epoll_wait`/`kqueue`/
`GetQueuedCompletionStatusEx`), run I/O callbacks, run check handles, then run
close callbacks[^3]. Getting the ordering right is why "just call the callback
later" behavior is predictable across platforms.

The **thread pool** is where the abstraction leaks by design. Filesystem calls
(`uv_fs_*`), DNS resolution (`uv_getaddrinfo`/`uv_getnameinfo`), and any work
you submit via `uv_queue_work` all execute on a shared, process-global pool —
four threads by default, sized by the `UV_THREADPOOL_SIZE` environment variable
which must be set *before* the pool is first used[^4]. The pool is global, not
per-loop: every loop in the process contends for the same threads.

Cross-thread communication is deliberately narrow. A loop is not thread-safe and
must be driven from exactly one thread. The only supported way to wake a loop
from another thread is `uv_async_t` / `uv_async_send`, and even that coalesces:
multiple sends before the callback runs may collapse into a single invocation,
so it is a wakeup signal, not a queue.

On Linux, libuv 1.45 added an **io_uring** backend that makes several
filesystem operations (read, write, fsync, and others) genuinely async instead
of thread-pool-backed, when the kernel supports it[^5]. This was a meaningful
architectural shift and also the source of the most disruptive regression in
recent memory (see Production Notes).

## Production Notes

**The thread pool is a shared bottleneck.** Because filesystem *and* DNS share
the same four-thread pool, a handful of slow operations — a `stat` on a hung NFS
mount, or a batch of blocking `getaddrinfo` lookups — can starve everything
else, including unrelated DNS in the same process. The symptom is latency that
has nothing to do with CPU or network. Raise `UV_THREADPOOL_SIZE` (max 1024),
but note it is per-process and set once at startup; you cannot resize it live.
This is the root cause behind many "Node.js is mysteriously slow under load"
reports.

**io_uring broke sandboxed containers.** When Node.js 20 shipped libuv 1.45
with io_uring enabled, workloads running under restrictive `seccomp` profiles
(gVisor, some Docker/Kubernetes defaults that block io_uring syscalls) saw
hangs and failures, because the blocked syscalls did not degrade gracefully[^5].
Later libuv releases added gating and a `UV_USE_IO_URING` control to disable it.
If you self-host Node or embed libuv in a sandboxed environment, verify io_uring
behavior against your seccomp policy before rolling out.

**Handle lifetime is manual and unforgiving.** You must call `uv_close` on a
handle and wait for its close callback before freeing the memory it lives in.
Freeing a handle that the loop still references is a classic use-after-free that
surfaces as intermittent crashes. There is no garbage collection; the ownership
contract is yours to enforce.

**`-fno-strict-aliasing` is recommended.** libuv uses ad-hoc struct
"inheritance" (casting between `uv_handle_t` and concrete handle types), which
can be unsafe under aggressive strict-aliasing optimizations. The project
recommends compiling consumers with `-fno-strict-aliasing`[^1].

**Platform parity is close but not total.** Signals are emulated on Windows;
there is no `fork`-style child model; a few filesystem-event and TTY behaviors
differ. On z/OS, System V semaphores and message queues persist after a process
exits unless the loop is closed, and may need manual `ipcrm` cleanup[^1].

## When to Use / When Not

**Use when:**
- You are writing a cross-platform async network service in C or C++ and want one API for sockets, timers, pipes, signals, and child processes.
- You are embedding an event loop into a language runtime or interpreter.
- You need real Windows *and* Unix support without maintaining two I/O backends.

**Avoid when:**
- You are building an application rather than infrastructure — reach for Node.js, Go, or an async runtime instead of raw libuv.
- You target Linux only and want maximum throughput — programming `io_uring` or `epoll` directly avoids the portability overhead.
- You want higher-level abstractions (coroutines, futures, typed sockets) — a C++ library like Asio fits better than libuv's callback-and-struct C API.

## Alternatives

- libevent — older C event-notification library; use it when you want a smaller, networking-focused loop and do not need Windows IOCP or a filesystem thread pool.
- enki/libev — minimal, high-performance Unix event loop (libuv historically wrapped it); use it when you are Unix-only and want the leanest possible core.
- chriskohlhoff/asio (Boost.Asio) — use it when you are in C++ and want coroutines, typed I/O objects, and RAII rather than a C callback API.
- axboe/liburing — use it when you are Linux-only and want direct io_uring with no portability layer.
- tokio-rs/tokio — use it when your project is in Rust rather than C/C++.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-1.0 | 2011 | Extracted from Node.js; wrapped libev + libeio on Unix, IOCP on Windows. |
| — | 2012 | libev/libeio dependency removed; native per-platform loop. |
| 1.0.0 | 2014 | First semantic-versioning release; stable-ABI-across-major commitment[^1]. |
| 1.45.0 | 2023 | io_uring backend added for Linux filesystem operations[^5]. |
| 1.46+ | 2023–2024 | io_uring gating / `UV_USE_IO_URING` control after seccomp-sandbox regressions[^5]. |

Note: libuv has remained on the 1.x line since 2014, preserving a stable ABI —
an unusually long major-version run driven by its role as a load-bearing
dependency of Node.js.

## References

[^1]: libuv README and project documentation. https://github.com/libuv/libuv and https://libuv.org/
[^2]: libuv historical context — extracted from Node.js, libev/libeio dependency later removed. https://docs.libuv.org/en/v1.x/design.html
[^3]: libuv design overview — "The I/O loop" iteration phases. https://docs.libuv.org/en/v1.x/design.html
[^4]: libuv thread pool work scheduling — `UV_THREADPOOL_SIZE`, shared by fs and DNS. https://docs.libuv.org/en/v1.x/threadpool.html
[^5]: libuv ChangeLog — io_uring support (1.45.0) and subsequent gating. https://github.com/libuv/libuv/blob/v1.x/ChangeLog

## Tags

c, asynchronous-io, event-loop, networking, cross-platform, epoll, kqueue, iocp, nodejs, io, thread-pool
