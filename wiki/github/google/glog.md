# google/glog

> C++ application-level logging via stream macros — Google's internal logging module, open-sourced. Now deprecated and archived.

[GitHub repo](https://github.com/google/glog) ·
[Official website](https://google.github.io/glog/) ·
[License: BSD-3-Clause](https://github.com/google/glog/blob/master/COPYING)

## Overview

glog is a C++ logging library built around stream-style macros (`LOG(INFO) << "msg"`) and a family of assertion macros (`CHECK_EQ`, `CHECK_NOTNULL`). It is the open-source release of the logging module Google uses internally, first published around 2008 and migrated to GitHub in 2015[^1]. For a decade it was a near-default choice for C++ projects that wanted severity levels, on-disk log files, verbose (`VLOG`) tracing, and crash stack traces without pulling in a larger framework — Caffe, Ceph, and many Google-adjacent C++ codebases depended on it.

**The most important fact about glog in 2026: it is deprecated and the repository is archived.** The maintainers announced end-of-life and archived the repo on 2025-06-30[^2]. The README now points users to two successors: `ng-log/ng-log`, a community fork that is API-compatible with a documented migration path, and Abseil's logging library, which Google maintains going forward. glog still builds and works, and its last release remains usable, but it will receive no further fixes, no new platform support, and no security patches.

The defining tension of glog was always its heritage: it encodes assumptions from Google's monolithic build and monolithic filesystem (log files written to `/tmp`, severity-named files with symlinks, `google::InitGoogleLogging` as a mandatory init step) that fit that environment better than a typical open-source application. It was reliable and battle-tested, but never opinion-free, and its macros collide loudly with other headers. Modern C++ has largely moved to header-only, fmt-based loggers.

## Getting Started

Via a package manager (vcpkg / conan / apt provide `glog`), or build from source with CMake:

```bash
git clone https://github.com/google/glog.git
cd glog
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
cmake --install build
```

Minimal program:

```cpp
#include <glog/logging.h>

int main(int argc, char* argv[]) {
  google::InitGoogleLogging(argv[0]);   // required; without it, logs go to stderr
  FLAGS_logtostderr = true;             // skip on-disk files, write to stderr

  LOG(INFO) << "starting up";
  int n = 42;
  LOG_IF(WARNING, n > 40) << "n is large: " << n;
  CHECK_GT(n, 0) << "n must be positive";   // aborts if the condition fails

  for (int i = 0; i < 100; ++i)
    LOG_EVERY_N(INFO, 10) << "tick " << google::COUNTER;   // logs every 10th

  VLOG(2) << "verbose detail, shown only when --v>=2";
  return 0;
}
```

CMake linkage: `target_link_libraries(app PRIVATE glog::glog)`.

## Architecture / How It Works

glog's public surface is macros, not functions. `LOG(severity)` expands to a temporary `LogMessage` object whose destructor flushes the accumulated stream to every registered sink. The four built-in severities are `INFO`, `WARNING`, `ERROR`, `FATAL`, in ascending order.

- **On-disk output.** By default glog writes one file per severity into a log directory (`--log_dir`, default `/tmp`), named `program.host.user.log.SEVERITY.timestamp.pid`, and maintains a `program.SEVERITY` symlink to the newest. Each severity file also contains all lower-severity messages. `--logtostderr` / `--alsologtostderr` redirect to the console.
- **`FATAL` and `CHECK`.** Logging at `FATAL` writes the message, flushes, and calls `abort()` — the process dies. The `CHECK`/`CHECK_EQ`/`CHECK_NOTNULL` family are runtime assertions that log `FATAL` on failure and are **not** compiled out in release builds (unlike `assert`). `DCHECK` variants are debug-only.
- **`VLOG` and verbosity.** `VLOG(n)` is conditional on a global verbosity level (`--v`) and per-module overrides (`--vmodule=file=level`), giving fine-grained trace control without recompilation.
- **Conditional/rate-limited macros.** `LOG_IF`, `LOG_EVERY_N`, `LOG_FIRST_N`, `LOG_EVERY_T` reduce log volume in hot paths.
- **Failure signal handler.** `google::InstallFailureSignalHandler()` installs handlers for `SIGSEGV`/`SIGABRT`/etc. that dump a stack trace on crash. Symbolized traces require libunwind (or the platform unwinder); without it, traces are addresses only.

Historically glog was tightly coupled to **gflags** for flag parsing. The 0.5.0 release (2021) reworked the build to make gflags optional and modernized the CMake to export the `glog::glog` target[^3]; when gflags is absent, flags are read from environment variables (`GLOG_logtostderr`, etc.) instead. Stack-trace and symbolization support is likewise a set of optional dependencies detected at configure time, which is why two builds of "the same" glog can behave differently on crash.

## Production Notes

**Disk fills silently.** glog does not rotate or delete old log files across process runs. `--max_log_size` caps the size of a single file (rolling to a new one within a run), but nothing cleans up yesterday's files. A long-lived service logging to the default `/tmp` will accumulate files until the volume fills. Operators are expected to wire external rotation/cleanup (logrotate, tmpwatch, a cron job) — this surprises people who assume the library manages its own retention.

**The `ERROR` macro collides with Windows headers.** `windows.h` defines `ERROR`, so `LOG(ERROR)` fails to compile in translation units that pull in the Windows SDK. The fix is `-DGLOG_NO_ABBREVIATED_SEVERITIES` (use `google::GLOG_ERROR`) and, in newer versions, the `GLOG_USE_GLOG_EXPORT` / export-header machinery. This is the single most common glog build issue on Windows.

**Init ordering is a real footgun.** Any `LOG` call before `google::InitGoogleLogging` goes to stderr with default settings; logging from static initializers (before `main`) therefore ignores your flags. Calling `InitGoogleLogging` twice also asserts.

**`CHECK` failures abort the process.** Because `CHECK` logs `FATAL`, a failed invariant terminates the program with `abort()` — appropriate for "cannot continue" conditions, dangerous when misused for recoverable errors. There is no exception path.

**Global-mutex contention.** The logging path serializes through a global lock. For very high-frequency logging on many threads this becomes a measurable bottleneck; hot paths should be gated behind `VLOG`/`LOG_EVERY_N` or moved off the critical path. This is a common reason projects migrate to spdlog's async, lock-light design.

**Migration off glog.** Since the project is archived, new work should target a successor. `ng-log` is deliberately API-compatible — for many codebases the migration is a header/namespace rename plus a build-system swap, documented by the fork[^2]. Abseil Logging (`absl/log`) is Google-maintained but has a different API surface, so it is a port rather than a drop-in.

## When to Use / When Not

**Use when:**
- You are maintaining an existing codebase already built on glog and are not ready to migrate — it still works.
- You depend on a third-party library (e.g. Caffe-era code) that requires the glog API and you want the reference implementation.

**Avoid when:**
- You are starting a new project — the archive/deprecation status alone disqualifies it; pick a maintained library.
- You need log rotation/retention, structured (JSON) logs, or async logging out of the box — glog provides none of these.
- You want header-only integration or `std::format`/fmt-style formatting rather than stream operators.
- You need first-class Windows support without macro workarounds.

## Alternatives

- gabime/spdlog — the most common modern C++ logger; header-only, fmt-based, async sinks, rotation built in. Use instead for essentially all new projects.
- ng-log/ng-log — the community continuation of glog itself, API-compatible with a migration guide. Use when you want to keep glog's exact API but on a maintained codebase.
- abseil/abseil-cpp — Abseil Logging (`absl/log`), Google-maintained. Use when you are already an Abseil user or want the Google-supported successor and can accept a different API.
- fmtlib/fmt — not a logger, but the formatting layer most modern loggers build on. Use alongside a logger, or when you only need formatting.
- SergiusTheBest/plog — small header-only logger. Use when you want minimal integration and few dependencies over glog's macros.

## History

| Version | Date | Notes |
|---------|------|-------|
| (Google Code) | ~2008 | Initial open-source release of Google's internal logging module. |
| GitHub import | 2015-02 | Repository created on GitHub[^1]. |
| 0.4.0 | 2019-03 | Bazel support improvements, CMake fixes. |
| 0.5.0 | 2021-05 | CMake modernization, `glog::glog` target, gflags made optional, C++14[^3]. |
| 0.6.0 | 2022-04 | Build/portability fixes, improved symbolization. |
| 0.7.0 | 2024-02 | Continued maintenance release; export-header changes. |
| Deprecation | 2025-06 | Project declared end-of-life; repository archived 2025-06-30[^2]. |

## References

[^1]: google/glog repository metadata (created 2015-02-23; archived; 7.3k stars, 2.1k forks; BSD-3-Clause; last push 2025-05-17). https://github.com/google/glog
[^2]: glog README deprecation notice — "This project is no longer maintained and will be archived on 2025-06-30." Points to ng-log (API-compatible) and Abseil Logging. https://github.com/ng-log/ng-log
[^3]: glog 0.5.0 release — CMake rework, optional gflags, `glog::glog` export target. https://github.com/google/glog/releases

## Tags

cpp, logging, c-plus-plus, observability, google, deprecated, archived, cmake, macros, stack-trace, application-logging
