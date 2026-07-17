# odygrd/quill

> Asynchronous, low-latency C++ logging library that defers all formatting and I/O to a single background thread.

[GitHub repo](https://github.com/odygrd/quill) ·
[Documentation](https://quillcpp.readthedocs.io) ·
[License: MIT](https://github.com/odygrd/quill/blob/master/LICENSE)

## Overview

Quill is a C++17 logging library built for one goal: make the cost of a log call on the
hot path as small and as predictable as possible[^1]. It is aimed at low-latency,
performance-critical software — trading systems, market-data handlers, game engines,
real-time and embedded-adjacent workloads — where a stray `printf` or a synchronous
`spdlog` write (hundreds of nanoseconds to microseconds) is unacceptable jitter.

The defining design choice is **deferred formatting**. A `LOG_INFO(...)` call does not
format its message. Instead it captures the format string, source location, and log level
as compile-time metadata, copies the runtime arguments into a per-thread lock-free queue,
and returns. A single backend thread later dequeues the arguments, formats them with the
bundled {fmt} library[^2], and writes to the sinks. The hot thread never touches a string,
never allocates for formatting, and never blocks on I/O in the common case.

The second differentiator is **timestamp-ordered output**. Because each thread has its own
queue, the backend drains all queues and emits records in timestamp order rather than
arrival order, so a multithreaded log reads chronologically without post-sorting. This is
the feature most users cite over the more popular gabime/spdlog. The tradeoff is that Quill
is not header-only in the usual sense — the backend must be compiled into exactly one
translation unit — and its API has churned across major versions (v2, v3, and v4 were each
substantial rewrites), so upgrades are rarely drop-in.

## Getting Started

Install via a package manager (vcpkg, Conan, Homebrew, Meson WrapDB, Conda, Bazel, xmake, nix):

```bash
vcpkg install quill      # or: brew install quill / conan install quill
```

```cpp
#include "quill/Backend.h"
#include "quill/Frontend.h"
#include "quill/LogMacros.h"
#include "quill/Logger.h"
#include "quill/sinks/ConsoleSink.h"

int main()
{
  quill::Backend::start();   // spins up the single backend thread

  quill::Logger* logger = quill::Frontend::create_or_get_logger(
    "root", quill::Frontend::create_or_get_sink<quill::ConsoleSink>("sink_id_1"));

  LOG_INFO(logger, "Hello from {}, run {}", "Quill", 42);
}
```

For the shortest path, `quill::simple_logger()` (from `quill/SimpleSetup.h`) returns a
console or file logger in one call. The macro API (`LOG_INFO`) is the lowest-latency path;
a macro-free function API (`quill::info(...)`) reads more like ordinary code but is slightly
slower because it loses some compile-time metadata capture.

## Architecture / How It Works

Quill splits cleanly into a **frontend** (what your threads call) and a **backend** (one
worker thread):

1. **Frontend.** `LOG_*` macros expand so the format string, file, line, and level become a
   `static constexpr` metadata struct — emitted once per call site, not per call. Only a
   pointer to that metadata plus the serialized arguments are pushed onto the calling
   thread's queue.
2. **Queue.** Each hot thread owns a single-producer/single-consumer lock-free ring buffer.
   No locks are shared between logging threads, so contention does not scale with thread
   count. Queues are **bounded or unbounded**, and **blocking or dropping**, chosen per logger.
3. **Backend.** One thread polls all queues, decodes the arguments, formats with {fmt},
   orders records by timestamp, and writes to sinks. Formatting and I/O cost lives entirely
   here, off every hot path.

Timestamps default to **`rdtsc`** (the CPU's time-stamp counter) read on the hot path, with a
backend `RdtscClock` that periodically resynchronizes the counter to wall-clock time.
`chrono` and custom clocks are also selectable — the custom-clock hook is what makes Quill
usable in deterministic simulations.

**Sinks** include console (with color), plain and rotating file sinks, JSON, and a
user-definable base. Additional facilities layered on the same backend: backtrace logging
(a ring buffer flushed on demand or on error), Mapped Diagnostic Context (thread-local
key/value pairs attached to lines), rate-limited macros (`LOG_*_LIMIT`), a crash-signal
handler that flushes pending logs, huge-pages support on Linux, wide-string handling on
Windows, and in-process metric publishing (Prometheus/StatsD/OpenTelemetry) routed through
the same backend.

## Production Notes

**Deferred formatting changes what is safe to log.** Because arguments are formatted *later*
on the backend thread, Quill copies them into the queue at call time. Trivially copyable
types and strings are copied by value and are safe. Passing a raw pointer or a reference to
a mutable object is a footgun: by the time the backend formats it, the pointee may have
changed or been destroyed. User-defined types that are not trivially copyable require a
custom `Codec` (serialization) or an eager `fmt::format` at the call site; otherwise they
won't compile or won't behave as expected.

**Queue mode is a real decision, not a default to ignore.** An *unbounded* queue that the
backend can't keep up with grows without limit — sustained log bursts become a memory-blowup
risk. A *bounded dropping* queue instead drops messages and increments a counter. Quill
exposes monitoring for dropped messages, queue reallocations, and blocked hot threads;
wire that telemetry up rather than assuming logs are lossless.

**The single backend thread is the serialization point.** Formatting cost is moved off the
hot path, but it does not disappear — under very high aggregate volume across many threads,
the backend's formatting + I/O becomes the bottleneck and back-pressures the queues. Sizing
sinks (async file writes, fast disks) matters at scale.

**`rdtsc` needs an invariant, synchronized TSC.** On hardware without invariant TSC, or on VMs
where the counter is unstable or unsynchronized across cores, timestamps can drift or
misorder despite the periodic resync. If TSC reliability is in doubt, switch the logger to
the `chrono` clock and accept the slightly higher hot-path cost.

**API churn across majors.** Major versions arrive often (v4 in mid-2024 through v12 in 2026)
and several were rewrites — the current `Backend::start()` / `Frontend::create_or_get_logger`
API was introduced in v4 and is incompatible with the v3 setup code. Pin a version, read the
changelog before bumping majors, and expect setup-code edits rather than a silent upgrade.

**The backend is a compiled TU.** Unlike header-only loggers, exactly one `.cpp` must pull in
the backend; the frontend headers (`Logger.h`, `LogMacros.h`) stay lightweight everywhere else.

## When to Use / When Not

**Use when:**
- Hot-path logging cost and jitter must be minimal and bounded (HFT, market data, real-time).
- You want chronologically ordered logs across threads without post-processing.
- You need structured JSON, backtrace-on-error, MDC, or in-process metrics on one backend.
- You can commit to copying/serializing your log arguments rather than logging live references.

**Avoid when:**
- A simple app where spdlog's convenience and stable API outweigh nanosecond latency.
- You need a strictly header-only logger with zero compiled translation units.
- You must log non-copyable objects formatted eagerly, or log-and-forget references.
- You want a rarely-changing API — Quill's major-version cadence implies periodic migration work.

## Alternatives

- gabime/spdlog — the popular default; far simpler API and larger ecosystem, synchronous by default with an optional async mode. Use when developer convenience beats hot-path latency.
- MengRao/fmtlog — same deferred-format, low-latency philosophy in a smaller single-header package. Use when you want comparable latency with fewer features and a lighter footprint.
- PlatformLab/NanoLog — research-grade minimum latency via binary logs that need an offline decoder. Use when you want the absolute floor and can post-process output.
- choll/xtr — another low-latency async C++ logger with competitive benchmarks. Use when you want a Quill-like design with a different, smaller feature set.
- Morgan-Stanley/binlog — binary, deferred logging for compact on-disk logs. Use when log volume and storage size dominate over human-readable output.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2020-03 | First release; async deferred-format design established[^3]. |
| 2.0.0 | 2022-05 | Major rewrite of the internals and API. |
| 3.0.0 | 2023-06 | Header reorganization; {fmt} bundling changes. |
| 4.0.0 | 2024-05 | Backend/Frontend split — the current setup API. |
| 5.0.0 | 2024-07 | Continued API refinement. |
| 7.0.0 | 2024-09 | — |
| 8.0.0 | 2025-01 | — |
| 10.0.0 | 2025-06 | — |
| 11.0.0 | 2025-11 | — |
| 12.0.0 | 2026-06 | Latest major line (12.1.0, 2026-07)[^3]. |

## References

[^1]: Quill README — "Asynchronous Low Latency C++ Logging Library." https://github.com/odygrd/quill
[^2]: {fmt} formatting library, on which Quill's type-safe API is built. https://github.com/fmtlib/fmt
[^3]: Quill releases and changelog. https://github.com/odygrd/quill/releases

## Tags

cpp, cpp17, logging, logging-library, low-latency, asynchronous, high-performance, deferred-formatting, fmtlib, cross-platform, backend-thread
