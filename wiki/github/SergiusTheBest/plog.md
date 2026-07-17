# SergiusTheBest/plog

> Header-only C++ logging library that fits its whole feature set into roughly a thousand lines of templates.

[GitHub repo](https://github.com/SergiusTheBest/plog) ·
[License: MIT](https://github.com/SergiusTheBest/plog/blob/master/LICENSE)

## Overview

Plog is a header-only C++ logging library first released in 2015[^1]. Its explicit design goal is to stay small — "slightly more than 1000 lines of code" per its own README — while still covering the cases most application loggers need: rolling files, colored console output, CSV output, multiple sinks, per-module instances, and Unicode. It positions itself as an alternative to heavier libraries (log4cxx, Boost.Log, glog) for projects that want a drop-in header rather than a build dependency.

The defining tension is minimalism versus feature creep. Plog leans on C++ templates instead of interface inheritance for its formatters, converters, and appenders, which keeps the compiled footprint small and avoids a linked binary, but pushes configuration into template parameters that are resolved at compile time. It deliberately does not require C++11, which broadens compiler support (it lists msvc, gcc, clang, mingw, icc, and C++Builder) at the cost of using older idioms internally. It is also synchronous by default — there is no built-in asynchronous/queued appender — so logging happens on the calling thread under a mutex.

As of this writing the project has ~2.6k stars and ~400 forks, MIT-licensed, and is still receiving commits (last push July 2026)[^2]. Release cadence has slowed to roughly one tagged version per year, which for a stable header-only utility reads as maturity rather than abandonment: the API surface has been essentially frozen since the 1.1.x line settled.

## Getting Started

Header-only — nothing to build or link. Copy the `include/` tree, add it as a submodule, or pull it via CMake:

```cmake
include(FetchContent)
FetchContent_Declare(plog
  GIT_REPOSITORY https://github.com/SergiusTheBest/plog
  GIT_TAG        1.1.11)
FetchContent_MakeAvailable(plog)
target_link_libraries(myproj plog::plog)   # sets include path too
```

Also available via vcpkg (`vcpkg install plog`), Conan, and NuGet.

```cpp
#include <plog/Log.h>
#include <plog/Initializers/RollingFileInitializer.h>

int main() {
    // maxSeverity, file, maxFileSize (bytes), maxFiles
    plog::init(plog::debug, "app.log", 1000000, 5);

    PLOGD << "debug via short macro";
    PLOG_INFO << "info via long macro";
    PLOG(plog::warning) << "warning via function-style macro";
    return 0;
}
```

Output format is inferred from the file extension: `.csv` selects `CsvFormatter`, anything else uses `TxtFormatter`. Log lines carry timestamp, severity, thread id, and `function@line` by default.

## Architecture / How It Works

Plog splits into four collaborating pieces, wired together mostly through templates:

- **Logger** — a singleton parameterized by an integer `instanceId` (default 0). It holds the max severity and a list of appenders. `plog::get<id>()` returns the instance (or `NULL` if uninitialized). Multiple loggers coexist by using distinct instance ids.
- **Record** — one log message plus its metadata (time, severity, tid, source location, the object's `this` pointer on MSVC). Built lazily by the logging macros.
- **Formatter** — turns a Record into text. Ships `TxtFormatter`, `CsvFormatter`, `FuncMessageFormatter`, `MessageOnlyFormatter`, each with UTC-time variants. Chosen as a template argument on the appender.
- **Appender** — the sink. `RollingFileAppender`, `ConsoleAppender`, `ColorConsoleAppender`, plus platform-specific ones: `AndroidAppender`, `EventLogAppender` (Windows Event Log), `DebugOutputAppender` (`OutputDebugString`), `ArduinoAppender`, and `DynamicAppender`. Converters (`UTF8Converter`, `NativeEOLConverter`) sit between formatter and file to handle encoding and line endings.

The macros are the ergonomic core. `PLOGD << x` expands to a severity check followed by stream construction, so the `<< x` right-hand side is only evaluated when the message would actually be logged — this is the "lazy stream evaluation" the README advertises, and it is what makes leaving debug logging in hot paths cheap. Macros come in three shapes (`PLOGD`, `PLOG_DEBUG`, `PLOG(sev)`), conditional `_IF` variants, per-instance `_` variants, and an `IF_PLOG(sev){...}` block guard for grouping expensive diagnostic work. Both `PLOG*` and legacy `LOG*` spellings exist; the `LOG*` set is prone to clashing with macros from other libraries and system headers, which is why the `PLOG` prefix became the default in 1.1.5.

Cross-module behavior is the sharp edge. Because the Logger is a singleton templated per instance, whether a `.dll`/`.so` shares the host's logger or gets its own depends on symbol visibility. Plog exposes `PLOG_GLOBAL`/`PLOG_LOCAL`/`PLOG_EXPORT`/`PLOG_IMPORT` macros to force the choice, and the defaults differ by OS (Windows defaults to local; Linux follows the compiler's `-fvisibility`). Chained loggers — where one Logger acts as an appender for another — are the sanctioned pattern for funneling a shared library's logs into the main binary.

## Production Notes

- **Synchronous under a mutex.** Every log call formats and writes on the calling thread while holding a lock. This is fine for typical application logging but is not a fit for high-throughput, latency-sensitive paths where a lock-free async queue (spdlog's async mode) matters. There is no built-in background flushing thread.
- **Cross-module logging is the top footgun.** The most common "logs disappear" or "duplicate logger" reports trace to visibility defaults across `.so`/`.dll` boundaries. On Linux, forgetting `PLOG_LOCAL` when you meant a per-module logger — or omitting `-fvisibility=hidden` reasoning — produces surprising sharing. Decide the sharing model explicitly for any plugin/shared-lib architecture.
- **Appender lifetime must be static.** Manual initialization passes a pointer to an appender you construct; if that appender is a local that goes out of scope, the logger dereferences freed memory. The README repeats "The appender lifetime should be static" for good reason.
- **Rolling is size/count based, not time based.** `maxFileSize` and `maxFiles` control rotation; there is no daily/hourly rotation primitive. If either is zero, rolling is off and the file grows unbounded. A pre-1.1.6 bug truncated file size accounting above 2 GB on Windows.
- **Unicode/UTF-8 on Windows was a long tail.** Files are written UTF-8 and the library targets the Utf8Everywhere approach, but full UTF-8 `char` encoding on Windows only landed in 1.1.10 (2023). Older versions leaned on wide strings for correct Windows Unicode. Console Unicode uses `WriteConsoleW`.
- **Binary size / disabling.** Since 1.1.6 logging can be compiled out to shrink binaries. Note that `plog/Init.h` was deliberately decoupled from `plog/Log.h` in 1.1.6, so custom-init code must include it explicitly — a silent breakage when upgrading older integrations.
- **Header-only means recompiles.** Changing a formatter or appender template argument is a compile-time decision; there is no runtime config file. Widespread inclusion of `plog/Log.h` couples build times to plog updates, so it is a natural precompiled-header candidate.

## When to Use / When Not

**Use when:**
- You want logging without adding a linked dependency or build step — copy headers and go.
- You target a wide compiler/platform matrix (including pre-C++11 toolchains, C++Builder, MinGW, embedded via Arduino/FreeRTOS/RTEMS).
- You need CSV log output, colored console, or Windows Event Log with minimal setup.
- Your logging volume is modest and synchronous writes are acceptable.

**Avoid when:**
- You need maximum throughput or non-blocking logging — a lock-free async logger is a better fit.
- You want runtime-reconfigurable logging via config files (log levels, sinks, patterns) without recompiling.
- You need structured/JSON logging out of the box (plog is line-oriented text/CSV; JSON means a custom formatter).
- You want rich pattern layouts and named-logger hierarchies in the log4j tradition.

## Alternatives

- gabime/spdlog — the default modern choice; async mode, fmt-based formatting, far higher throughput. Use instead when performance or structured sinks matter and a compiled dependency is acceptable.
- fmtlib/fmt — not a logger, but pairs with or underlies spdlog; use when you primarily need fast type-safe formatting and can build logging on top.
- google/glog — Google's C++ logging with `CHECK`/`VLOG` macros and stack traces. Use instead when you want Google-style fatal-check semantics.
- emilk/loguru — another small, header-lean logger with callback sinks and nicer default formatting. Use instead when you want minimalism plus built-in fatal handling and callbacks.
- apache/logging-log4cxx — full log4j-style config-file-driven hierarchy. Use instead when runtime configuration and appender hierarchies are requirements.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2015-05-19 | Initial public release[^1]. |
| 1.0.1 | 2015-11-01 | ColorConsoleAppender; MinGW-w64 fixes. |
| 1.1.0 | 2016-11-20 | Binary-compatible Record interface (not mixable with 1.0.x in chained mode). |
| 1.1.4 | 2018-03-26 | FreeBSD, RTEMS, Intel C++ Compiler support. |
| 1.1.5 | 2019-10-21 | `PLOG`-prefixed macros to avoid `LOG` clashes; UTC time; printf-style; modern CMake. |
| 1.1.6 | 2022-02-06 | Disable-logging option; `std::filesystem::path`; decouple `Init.h` from `Log.h`. |
| 1.1.7 | 2022-06-09 | Hex/ASCII dumpers; std-container printing; `find_package`; license changed to MIT. |
| 1.1.9 | 2022-12-16 | Truncate-with `>`; `override` specifiers; add/remove appenders API. |
| 1.1.10 | 2023-08-20 | UTF-8 `char` encoding on Windows (Utf8Everywhere)[^3]. |
| 1.1.11 | 2025-08-11 | FreeRTOS support. |

## References

[^1]: Plog README, "Version history" — Version 1.0.0, 19 May 2015. https://github.com/SergiusTheBest/plog#version-history
[^2]: GitHub REST API, `repos/SergiusTheBest/plog` — 2,564 stars, 406 forks, MIT, last push 2026-07-08 (fetched 2026-07-17). https://github.com/SergiusTheBest/plog
[^3]: Plog README, "Version 1.1.10 (20 Aug 2023)". https://github.com/SergiusTheBest/plog#version-history

## Tags

cpp, c-plus-plus, logging, logger, header-only, cross-platform, library, no-dependencies, template-based, embedded
