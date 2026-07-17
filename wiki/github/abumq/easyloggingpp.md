# abumq/easyloggingpp

> Single-header C++ logging library with macro-driven configuration — feature-complete, widely embedded, and archived since 2018.

[GitHub repo](https://github.com/abumq/easyloggingpp) ·
[License: MIT](https://github.com/abumq/easyloggingpp/blob/master/LICENCE)

## Overview

Easylogging++ (often written `easylogging++` or `el`) is a C++ logging library distributed as a single header plus one companion source file. It was written by Majid Q. ([@abumq](https://github.com/abumq), previously amrayn/mkhan3189) and developed actively from 2012 to 2018[^readme]. The repository is now **archived** and read-only: no issues, no pull requests, no releases[^api]. The maintainer's own README recommends `spdlog` or another maintained library for new projects[^readme].

Its defining trait is that almost everything happens through preprocessor macros and global state. You call `INITIALIZE_EASYLOGGINGPP` exactly once per program to define the library's `extern` globals, then log with `LOG(INFO) << "..."`. Configuration, verbosity, performance timers, and crash handling are all reached through macros rather than an object you thread through your code. This makes trivial adoption genuinely trivial — two files, one macro, done — and makes the library awkward to sandbox, test in isolation, or run more than one independent logger configuration inside.

The design goal was zero external dependencies: threading, date/time formatting, and configuration parsing are all built in rather than pulled from Boost or Qt[^readme]. That self-containment is why it spread into embedded and desktop projects, and also why some of its subsystems (its own thread wrappers, its own config file format) feel dated next to modern alternatives built on `fmt` and `std::` primitives.

## Getting Started

Vendor the two files (`easylogging++.h` and `easylogging++.cc`) into your project, or install via vcpkg (`vcpkg install easyloggingpp`)[^readme].

```c++
// main.cc
#include "easylogging++.h"

INITIALIZE_EASYLOGGINGPP   // once per application, in the TU with main()

int main(int argc, char* argv[]) {
    START_EASYLOGGINGPP(argc, argv);   // optional: enables --v / vmodule args
    LOG(INFO) << "first log line";
    LOG(WARNING) << "value = " << 42;
    return 0;
}
```

```
g++ main.cc easylogging++.cc -o prog -std=c++11
```

C++11 is required; the older `v8.91` line is the last to support C++98/C++03[^readme].

Runtime configuration can be driven from a file in the library's own format:

```
* GLOBAL:
   FORMAT               = "%datetime %level %msg"
   FILENAME             = "/tmp/app.log"
   TO_STANDARD_OUTPUT   = true
   MAX_LOG_FILE_SIZE    = 2097152   ## 2MB, then truncate
* DEBUG:
   FORMAT               = "%datetime{%d/%M} %func %msg"
```

```c++
el::Configurations conf("app.conf");
el::Loggers::reconfigureAllLoggers(conf);
```

## Architecture / How It Works

The library is one large header defining the `el` namespace plus a set of top-level macros. `INITIALIZE_EASYLOGGINGPP` instantiates the static storage (logger registry, default configuration, the global lock), which is why it must appear in exactly one translation unit — duplicate it and you get linker errors, omit it and you get undefined symbols.

Levels are `Global, Trace, Debug, Fatal, Error, Warning, Info, Verbose, Unknown`. By deliberate design the library is **not hierarchical** by default — enabling `INFO` does not enable `WARNING` — so you turn each level on or off explicitly unless you opt into `LoggingFlag::HierarchicalLogging`[^readme]. `Verbose` is a separate axis driven by a numeric level (`VLOG(2)`) and the `--v` / `-vmodule` command-line arguments.

Configuration has three front doors: a config *file* in a bespoke `* LEVEL:` / `KEY = "VALUE"` format, the `el::Configurations` class, or inline strings. Format is a pattern string of specifiers (`%datetime`, `%level`, `%logger`, `%func`, `%loc`, `%msg`) with sub-second precision and custom date formats. Loggers are looked up by string id (`CLOG(INFO, "network")`), and IDs are created on first reference.

Bundled extras, each gated behind a compile-time `ELPP_FEATURE_*` or `ELPP_*` macro: performance tracking (`TIMED_SCOPE`, `TIMED_FUNC`), a default crash handler that prints a stack trace, `CHECK_*` assertion macros modeled on glog, STL/`std::` container logging, syslog output, and adapters for logging Qt / Boost / wxWidgets types. Custom sinks are possible through a `LogDispatchCallback`, which is the intended extension point for shipping logs to a network or a database.

## Production Notes

- **Not thread-safe unless you ask.** Concurrent logging is only guarded when the library is compiled with `ELPP_THREAD_SAFE` defined. Build without it and multi-threaded logging can interleave or corrupt output. This is the single most common surprise in real deployments.
- **It installs signal handlers by default.** The default crash handler traps `SIGABRT`, `SIGSEGV`, `SIGILL`, `SIGFPE`, and friends to print a stack trace. In a process that already manages its own signals (a game engine, a supervised daemon), this silently competes with your handler. Disable with `ELPP_DISABLE_DEFAULT_CRASH_HANDLING`.
- **It writes a log file whether you asked or not.** The default configuration logs to a file (historically `myeasylog.log`) in the working directory. Set `ELPP_NO_DEFAULT_LOG_FILE`, or reconfigure `To_File`/`Filename` up front, to avoid stray files.
- **Global state, one configuration tree.** Everything routes through process-global singletons initialized by `INITIALIZE_EASYLOGGINGPP`. Running two components with genuinely independent logging configs in one process, or unit-testing logging behavior in isolation, is difficult by construction.
- **Macro-heavy surface.** Because features are macros, they interact with your build's own macros and with unity builds; name collisions and "works in debug, silent in release" (`DEBUG`/`NDEBUG`-gated levels) issues do occur.
- **No upgrades coming.** Archived means security or portability fixes for newer compilers/toolchains will not land upstream. You are maintaining any patches in your fork. `303` open issues sit frozen at archive time[^api].

## When to Use / When Not

**Use when:**
- You have an existing project already on Easylogging++ and it works — there is no urgency to rip it out.
- You need a drop-in logger for a small tool or embedded target with no dependency manager, and you value "two files, one macro" over long-term maintenance.
- You specifically want its built-in performance tracking or config-file-driven format without adding `fmt`.

**Avoid when:**
- You are starting a new project — the maintainer explicitly points new users to spdlog[^readme].
- You need low-latency asynchronous logging as a first-class, well-tested path (its async support was long marked experimental).
- You want per-component isolated logger instances, easy testability, or a dependency that will keep pace with new C++ standards and compilers.

## Alternatives

- gabime/spdlog — the maintainer's own recommendation; `fmt`-based, fast async, actively maintained. Default choice for new C++ logging.
- google/glog — similar `LOG(INFO)`/`CHECK` macro style that Easylogging++ echoes; use when you want the Google-ecosystem conventions.
- odygrd/quill — use when low-latency asynchronous logging with a real back-thread is the priority.
- SergiusTheBest/plog — use when you want a minimal header-only logger with almost no surface area.
- emilk/loguru — use when you want simple, readable stderr logging plus stack traces without a config-file format.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2012-09-23 | Repository created on GitHub[^api]. |
| v8.91 | — | Last line supporting C++98/C++03[^readme]. |
| v9.89 | — | Last header-only release; later versions require `easylogging++.cc`[^readme]. |
| v9.97.1 | — | Version the current documentation targets[^readme]. |
| (active dev ends) | 2018 | Maintainer states active development ran 2012–2018[^readme]. |
| archived | 2025-07-06 | Repository set read-only; last recorded push[^api]. |

## References

[^api]: GitHub REST API, `repos/abumq/easyloggingpp` — archived: true, stars 3941, forks 944, open issues 303, license MIT, created 2012-09-23, last push 2025-07-06. Fetched 2026-07-17. https://github.com/abumq/easyloggingpp
[^readme]: Easylogging++ README (archive notice + v9.97.1 documentation), abumq/easyloggingpp, master branch. https://github.com/abumq/easyloggingpp/blob/master/README.md

## Tags

cpp, c-plus-plus-11, logging, logging-library, single-header, header-only, crash-handler, thread-safety, archived, macros
