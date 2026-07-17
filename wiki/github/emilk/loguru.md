# emilk/loguru

> A two-file C++11 logging library with stack traces, assertion macros, and grep-friendly line output — from the author of egui.

[GitHub repo](https://github.com/emilk/loguru) ·
[Documentation](https://emilk.github.io/loguru/index.html) ·
[License: Unlicense](https://github.com/emilk/loguru/blob/master/LICENSE)

## Overview

Loguru is a small C++ logging library written and maintained primarily by Emil Ernerfeldt, better known for the `egui` immediate-mode GUI toolkit[^1]. It ships as exactly two files — `loguru.hpp` and `loguru.cpp` — with a header that has *no `#include`s* of its own, a deliberate choice to keep compile times low in projects that log frequently[^2]. It targets C++11 and has no third-party dependencies unless you opt into fmtlib formatting.

The library positions itself as a lighter, mostly drop-in alternative to Google's glog: it offers the same `LOG_F(INFO, ...)` / `CHECK_F(...)` vocabulary, printf-style *and* stream-style (`LOG_S(INFO) << ...`) formatting, verbosity levels, and stack traces on crash. Its distinguishing features are indentation *scopes* that visually nest related log lines, `ERROR_CONTEXT` values that are captured cheaply and only printed if a crash occurs, and user-installable callbacks so an application (a game, say) can draw severe messages on screen or throw instead of aborting.

The defining tension is that Loguru is a single-author convenience library, not an infrastructure-grade logging framework. It leans heavily on process-global mutable state, installs signal handlers by default, and is line-oriented text only — there is no built-in structured/JSON output, no sink abstraction, and no async back-end thread. For a personal or mid-sized project that wants readable, greppable logs with crash diagnostics in two files, that is exactly the point. For a high-throughput service or a library meant to drop into someone else's process, the global-state and signal-handler assumptions become the friction.

## Getting Started

There is no package to install — you vendor the two files. Compile `loguru.cpp` (or `#include` it from one of your own `.cpp` files) and link the platform libraries:

```bash
g++ -std=c++11 main.cpp loguru.cpp -lpthread -ldl -o app
```

```cpp
#include <loguru.hpp>

int main(int argc, char* argv[])
{
    // Parses -v from argv and timestamps the start of the log.
    loguru::init(argc, argv);

    // Full-verbosity append log + a readable truncated log:
    loguru::add_file("everything.log", loguru::Append, loguru::Verbosity_MAX);
    loguru::add_file("readable.log",   loguru::Truncate, loguru::Verbosity_INFO);

    LOG_SCOPE_F(INFO, "Starting work");      // indents everything in this block
    LOG_F(INFO, "Hungry for some %.3f!", 3.14159);
    LOG_IF_F(WARNING, argc < 2, "No arguments given");

    FILE* fp = fopen("data.bin", "rb");
    CHECK_F(fp != nullptr, "Failed to open file");   // aborts + stack trace on fail
    return 0;
}
```

CMake integration is supported via `add_subdirectory()`, `FetchContent`, or `find_package()`[^3]. For `std::ostream`-style logging, `#define LOGURU_WITH_STREAMS 1` before including the header (this pulls in `<sstream>`).

## Architecture / How It Works

Loguru is built around a small set of process-global variables and a global list of registered outputs. `loguru::init()` records the start time, optionally reads a `-v` verbosity flag from `argv`, sets the main thread name, and — unless disabled — installs signal handlers for `SIGABRT`, `SIGSEGV`, `SIGBUS`, and friends so a crash prints a cleaned-up stack trace. Verbosity is a signed integer: negatives are `WARNING`/`ERROR`/`FATAL`, `INFO` is 0, and higher positive numbers are progressively noisier `VLOG` levels you can filter after the fact.

The macro layer is where most of the surface lives. `LOG_F` and `VLOG_F` do printf-style logging (format-checked at compile time on GCC/Clang); `LOG_S`/`VLOG_S` do stream-style. The `CHECK_*_F` family (`CHECK_EQ_F`, `CHECK_GT_F`, …) are assertions that print the operand values on failure and are marked `noreturn` on abort so the optimizer and static analyzer understand control flow. Every macro has a `D`-prefixed debug-only variant that compiles out under `NDEBUG`.

Output is fan-out: every enabled log line is written to `stderr` (with ANSI color on capable terminals) and to each registered file whose verbosity threshold it meets. Writes are serialized by a mutex, making logging thread-safe at the cost of contention under heavy concurrent logging. Flushing is configurable through `loguru::g_flush_interval_ms`: set it to 0 for unbuffered writes that survive a hard crash, or to ~100 ms to let a background thread batch flushes for higher throughput. fmtlib support is compiled in with `#define LOGURU_USE_FMTLIB 1`, which adds fmt as a link/include dependency (or header-only via `FMT_HEADER_ONLY`).

Stack traces on POSIX are produced with `dlopen`/`dladdr` and `backtrace` (hence `-ldl`), then demangled and lightly cleaned so `std::__1::basic_string<...>` collapses to `std::string`. Frames are printed in chronological order with the most relevant last[^4]. Windows support exists but its traces are less complete than POSIX.

## Production Notes

**Signal handlers are the biggest footgun.** By default `loguru::init` installs handlers that walk and print a stack trace from inside the signal handler. That code path is not strictly async-signal-safe (it allocates and takes locks), so in a genuinely corrupted crash it can deadlock or fault again. It also *conflicts with other crash machinery* — Breakpad/Crashpad, Sentry native, Google Test's own death-test handlers, or a JVM/Go host — because whoever installs last wins. If you use a dedicated crash reporter, disable Loguru's handlers via the `SignalOptions` passed to `init`.

**It is global-state, not injectable.** Verbosity, color, flush interval, the fatal handler, and the file list are all process-global singletons. Two libraries in one process cannot each have their own Loguru configuration, and `loguru::init` consuming `-v` from `argv` surprises applications that parse their own flags. This is fine for an application owning `main()`; it is a poor fit for a *library* you expect others to embed.

**No structured logging.** Output is human-readable text lines with a fixed prefix (date, uptime, thread, file:line, level, indentation). There is no JSON/logfmt mode and no sink abstraction. Shipping to Elasticsearch/Loki/Datadog means either parsing the text or installing a `loguru::add_callback` handler and serializing yourself.

**Maintenance is quiet.** The repository is not archived, but activity is low — the last push was mid-2024 and there are roughly 90 open issues[^5]. Treat it as stable-and-slow rather than actively evolving; do not expect fast turnaround on platform-specific trace bugs. Because you vendor two files, an unmaintained upstream is low-risk: you already own your copy.

**Performance is adequate, not specialized.** The README reports single-digit-microsecond per-line costs and rough parity-to-2x versus glog on the author's hardware[^2] — good enough for most applications, but well behind purpose-built low-latency loggers that keep formatting off the hot path. Under many threads all logging serially, the single output mutex becomes the bottleneck.

## When to Use / When Not

**Use when:**
- You own the application's `main()` and want readable, greppable logs plus crash stack traces in two vendored files.
- You are migrating off glog and want a lighter, similar API.
- You value fast compile times and minimal dependencies (the header includes nothing).
- Scopes, `CHECK_*` assertions, and `ERROR_CONTEXT` crash diagnostics match how you debug.

**Avoid when:**
- You are writing a *library* meant to embed in arbitrary hosts — the global state and default signal handlers will fight the host.
- You need structured/JSON logs, pluggable sinks, or log shipping out of the box.
- You have a latency-critical hot path where async, back-end-thread logging matters.
- You already run a native crash reporter (Crashpad/Sentry) and don't want handler conflicts.

## Alternatives

- gabime/spdlog — the de facto C++ logging standard; fmt-based, async option, many sinks. Use instead when you want sinks, rotation, and an actively maintained ecosystem.
- google/glog — the library Loguru mirrors. Use when you're in a Google/Bazel-oriented codebase that already expects it.
- odygrd/quill — asynchronous, back-end-thread design for very low front-end latency. Use when per-call logging cost on a hot path is critical.
- SergiusTheBest/plog — header-only and minimal. Use when you want the smallest possible drop-in with fewer features.
- fmtlib/fmt — not a logger, but the formatting layer spdlog and others build on. Use directly when you only need formatting and will handle sinks yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-03-22 | First commit; two-file printf-style logger with stack traces[^1]. |
| 1.x | 2015–2018 | printf-style logging, verbosity, scopes, glog-style CHECK macros. |
| 2.x | 2018 onward | API reorganization; stream-style logging, fmtlib support, CMake integration[^3]. |
| — | 2024-07-01 | Last commit as of this writing; low ongoing activity[^5]. |

## References

[^1]: Loguru repository and README, Emil Ernerfeldt. https://github.com/emilk/loguru
[^2]: "No includes in loguru.hpp" and performance notes, Loguru README. https://github.com/emilk/loguru#no-includes-in-loguruhpp
[^3]: CMake integration example, Loguru repository. https://github.com/emilk/loguru/blob/master/loguru_cmake_example/CMakeLists.txt
[^4]: "Upside-down stack traces," Yeller (linked from README on stack-trace ordering). http://yellerapp.com/posts/2015-01-22-upside-down-stacktraces.html
[^5]: Repository metadata (last push 2024-07-01, ~90 open issues), GitHub API for emilk/loguru. https://github.com/emilk/loguru

## Tags

cpp, logging, c-plus-plus, stack-trace, header-library, assertions, debugging, cross-platform, unlicense, glog-alternative
