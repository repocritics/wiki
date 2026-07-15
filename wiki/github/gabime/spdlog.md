# gabime/spdlog

> Header-only (or optionally compiled) C++ logging library built on top of fmt, trading a small compile-time cost for very low runtime logging overhead.

[GitHub repo](https://github.com/gabime/spdlog) ·
[Wiki / docs](https://github.com/gabime/spdlog/wiki) ·
[License: MIT](https://github.com/gabime/spdlog/blob/v1.x/LICENSE)

## Overview

spdlog is a C++11 logging library first released as a 1.0 in 2018[^1], though the repository dates to 2014. It occupies a specific niche: applications that want structured, formatted, leveled logging with predictable latency, without pulling in a heavyweight framework like Boost.Log or a config-file-driven system like log4cxx. Its API is a thin set of free functions (`spdlog::info("...", args)`) plus named logger objects retrieved from a global registry.

The defining design choice is that spdlog is built directly on the fmt formatting library[^2]. Format strings use Python/fmt-style `{}` placeholders rather than printf's `%d`, and this dependency is both the library's biggest asset (fast, type-safe, compile-time-checkable formatting) and its most common source of build pain. spdlog bundles a copy of fmt under `include/spdlog/fmt/bundled/`, but can also be pointed at a system fmt via `SPDLOG_FMT_EXTERNAL`. Getting that switch wrong — or mixing a compiled spdlog built against one fmt with a translation unit that includes another — is the single most frequent integration failure in the ecosystem.

The library is mature and still maintained: the current stable line is v1.x (latest v1.17.0, January 2026), and a v2.x line is in development on separate branches. Releases arrive a few times per year and the API has been stable for years, so "actively maintained" here means steady maintenance rather than rapid change.

## Getting Started

```bash
# vcpkg
vcpkg install spdlog
# Debian/Ubuntu
sudo apt install libspdlog-dev
# Homebrew
brew install spdlog
```

Header-only requires only a C++11 compiler and copying `include/spdlog` into your tree. The compiled library (recommended) drastically cuts per-TU compile times:

```cpp
#include "spdlog/spdlog.h"
#include "spdlog/sinks/rotating_file_sink.h"

int main() {
    spdlog::info("Welcome to spdlog! arg={}", 42);

    // 5 MB per file, keep 3 rotated files, thread-safe (_mt)
    auto logger = spdlog::rotating_logger_mt("app", "logs/app.txt",
                                             1024 * 1024 * 5, 3);
    logger->set_level(spdlog::level::debug);
    logger->set_pattern("[%Y-%m-%d %H:%M:%S.%e] [%n] [%^%l%$] %v");
    logger->warn("padded: {:08d}", 12);
    spdlog::shutdown();   // flush + drop registered loggers
}
```

## Architecture / How It Works

The core model is **logger → sinks → formatter**. A `logger` has a name, a level, and one or more `sink` objects. Each sink owns its own formatter and its own level, so a single logger can, for example, write everything to a file while showing only warnings on a colored console. Sinks come in single-threaded (`_st`) and thread-safe (`_mt`) variants; the `_mt` versions take a mutex per log call, the `_st` ones assume you never touch them from more than one thread.

Formatting is delegated to fmt. The `pattern_formatter` compiles a pattern string (e.g. `%^%l%$` for colored level) into a list of flag formatters once, then reuses it. Custom pattern flags and custom sinks are the two documented extension points.

Named loggers live in a process-global `registry` guarded by a mutex. `spdlog::get("name")`, `spdlog::register_logger`, and the level-setting helpers all funnel through it. This is convenient but makes the registry a shared-state and lock-contention point in programs that create or look up loggers frequently on the hot path — the common advice is to grab the `shared_ptr` once and reuse it.

**Async logging** is opt-in (`#include "spdlog/async.h"`). An `async_logger` pushes a copy of each message into a bounded MPMC queue served by a shared thread pool (`init_thread_pool(queue_size, n_threads)`), so the calling thread returns after the enqueue rather than after the write. The overflow policy is either `block` (back-pressure the caller) or `overrun_oldest` (silently drop the oldest queued message). Compile-time level gating (`SPDLOG_ACTIVE_LEVEL` + the `SPDLOG_TRACE`/`SPDLOG_DEBUG` macros) removes filtered calls entirely from release builds.

## Production Notes

- **fmt version coupling is the top footgun.** If you build the compiled `spdlog` library against its bundled fmt and then include an external, different fmt version in your own code, you get ODR/ABI mismatches that surface as link errors or crashes. Decide project-wide: either everything uses bundled fmt, or everything sets `SPDLOG_FMT_EXTERNAL` against one pinned fmt.
- **Header-only inflates compile times.** Because spdlog pulls fmt into every translation unit that includes it, large codebases see meaningful build-time cost. The compiled library (`SPDLOG_COMPILED_LIB`, the default when you build via CMake) is the standard mitigation and is explicitly recommended in the README.
- **Async can lose messages on exit.** Queued messages are only guaranteed written if you flush/shut down cleanly (`spdlog::shutdown()` or letting the registry destruct). A hard `_exit` or crash drops whatever is still in the queue. Choose `block` vs `overrun_oldest` deliberately: `block` can stall producer threads under sustained load; `overrun_oldest` drops data without an error.
- **MDC (mapped diagnostic context) is not supported in async mode** because it relies on thread-local storage that the worker thread doesn't share.
- **Flushing.** Sinks do not flush on every message by default (for speed). Use `flush_on(level)` or `flush_every(...)` so logs survive a crash — but note `flush_every` is only safe with `_mt` loggers.
- **Global/default logger and static init.** The default logger and registry are process globals; logging from static initializers or destructors runs into initialization-order issues.
- **Windows wide-char/UTF-8** handling (`SPDLOG_WCHAR_TO_UTF8_SUPPORT`, `SPDLOG_WCHAR_FILENAMES`) is a compile-time decision that must match across the whole build.

## When to Use / When Not

**Use when:**
- You want fast, formatted, leveled logging in C++11+ without a framework or runtime config files.
- You need multiple sinks (rotating/daily files, console, syslog, Windows event log, Qt) with per-sink levels and formats.
- You want optional async logging with explicit back-pressure/drop semantics.
- You already use fmt, or are happy to standardize on fmt-style `{}` formatting.

**Avoid when:**
- You need the lowest possible tail latency in the hot path — dedicated low-latency loggers (Quill, NanoLog) defer formatting off the critical thread more aggressively.
- You want zero external formatting dependency and printf-style `%d` semantics.
- You need runtime-reconfigurable logging from external config files (log4cxx / Boost.Log territory).
- Header-only compile-time cost in a very large codebase is a hard constraint and you cannot use the compiled build.

## Alternatives

- odygrd/quill — use instead when you need lower and more predictable tail latency; it pushes almost all work (including formatting) to a background thread.
- fmtlib/fmt — use instead when you only need formatting, not logging; spdlog is a layer on top of it.
- google/glog — use instead when you want simple `LOG(INFO)` macro-style logging with stack traces and are fine with an older, less format-rich API.
- boostorg/log — use instead when you're already deep in Boost and want attributes/filters/sinks configured at runtime.
- PlatformLab/NanoLog — use instead when raw throughput and nanosecond-scale call cost dominate and you can tolerate a post-processing step to decode logs.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo start) | 2014-11 | Initial development as a header-only logger. |
| 1.0.0 | 2018-08-05 | First stable 1.0; API stabilized[^1]. |
| 1.4.0 | 2019-09-21 | Adopted fmt as the formatting backend more deeply; bundled fmt. |
| 1.9.0 | 2021-07-20 | Load levels from env/argv (`spdlog::cfg`). |
| 1.11.0 | 2022-11-02 | Backtrace, stopwatch, and formatting refinements. |
| 1.15.0 | 2024-11-09 | MDC support, continued fmt upgrades. |
| 1.16.0 | 2025-10-11 | Maintenance and fmt compatibility updates. |
| 1.17.0 | 2026-01-04 | Latest stable on the v1.x line. |

## References

[^1]: spdlog releases — v1.0.0 published 2018-08-05. https://github.com/gabime/spdlog/releases
[^2]: fmt formatting library (fmtlib/fmt), spdlog's formatting backend. https://github.com/fmtlib/fmt
[^3]: spdlog wiki — sinks, async logging, and custom formatting. https://github.com/gabime/spdlog/wiki

## Tags

cpp, cpp11, logging, header-only, async, fmt, library, observability, structured-logging, cross-platform
