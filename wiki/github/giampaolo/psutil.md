# giampaolo/psutil

> Cross-platform Python library for reading process and system metrics (CPU, memory, disk, network, sensors) through one API.

[GitHub repo](https://github.com/giampaolo/psutil) ·
[Documentation](https://psutil.readthedocs.io/) ·
[License: BSD-3-Clause](https://github.com/giampaolo/psutil/blob/master/LICENSE)

## Overview

psutil (process and system utilities) is a Python library, started by Giampaolo Rodolà, that exposes information normally scattered across platform-specific tools — `ps`, `top`, `free`, `netstat`, `iotop`, `lsof` — behind a single portable API[^1]. It covers running processes and system-wide utilization: CPU times and percentages, virtual and swap memory, disk partitions and I/O, network connections and counters, hardware sensors, users, and boot time. It is one of the most-depended-on packages on PyPI, reporting hundreds of millions of downloads per month and use by hundreds of thousands of public repositories[^2].

The defining tension is that psutil presents a *uniform* API over deeply *non-uniform* operating systems. The same `psutil.Process.memory_info()` call maps to `/proc` parsing on Linux, Mach calls on macOS, sysctl on the BSDs, and NtQuerySystemInformation-family calls on Windows. Most of the surface is implemented in per-platform C extension modules, so psutil is a compiled package, not pure Python. This is what makes it fast and accurate, and also what makes installation and cross-platform behavior the two areas where users get surprised.

The project is actively maintained after more than a decade — with a recent push in mid-2026 — and treats API stability seriously; the last sweeping breaking change was the 2.0 redesign in 2014. It is BSD-3 licensed, which is part of why it appears as a transitive dependency in so much monitoring, profiling, and infrastructure tooling.

## Getting Started

```bash
pip install psutil
```

Wheels are published for common CPython versions on Linux, macOS, and Windows, so most users install without a compiler. Building from source (older platforms, unusual architectures, free-threaded builds) needs a C toolchain and Python development headers.

```python
import psutil

# System-wide
print(psutil.cpu_percent(interval=1))          # blocking 1s sample, e.g. 12.3
print(psutil.virtual_memory().percent)         # 37.6
print(psutil.disk_usage("/").free)             # bytes free on the root fs

# Per-process
p = psutil.Process()                           # current process by default
with p.oneshot():                              # batch syscalls for this block
    print(p.name(), p.pid)
    print(p.memory_info().rss)                 # resident set size, bytes
    print(p.cpu_percent(interval=0.1))

# Iterate all processes cheaply
for proc in psutil.process_iter(["pid", "name"]):
    print(proc.info["pid"], proc.info["name"])
```

Most functions return named tuples (e.g. `svmem`, `sdiskusage`, `pmem`), so results are both index- and attribute-addressable.

## Architecture / How It Works

psutil is a thin, well-documented Python layer over a set of platform C extensions:

- `_psutil_linux`, `_psutil_osx`, `_psutil_windows`, `_psutil_bsd`, `_psutil_sunos`, `_psutil_aix` — compiled modules that call native OS interfaces.
- Pure-Python platform files (`_pslinux.py`, `_psosx.py`, `_pswindows.py`, …) that parse `/proc`, sysfs, or wrap the C module, and normalize results into the shared named-tuple types.
- A public `psutil/__init__.py` that dispatches to the right platform module at import time and defines the `Process` class.

`Process` objects are lazy handles around a PID. Each accessor (`.cpu_times()`, `.memory_info()`, `.open_files()`) is a fresh syscall, so reading ten fields makes ten calls. `Process.oneshot()` is a context manager that caches several kernel reads for the duration of the block, cutting redundant syscalls when you sample many fields at once — a real performance lever on process-heavy loops.

Two design decisions shape everyday use. First, **percentage functions are stateful over time**: `cpu_percent()` and `Process.cpu_percent()` compute a delta since the previous call. The first call returns `0.0`; you either pass a blocking `interval` or call twice with a gap. Second, **PID reuse is handled explicitly**: psutil records process creation time and raises `NoSuchProcess` if a PID has been recycled, rather than silently returning another process's data.

Not every function exists on every platform, and some return platform-specific extra fields. `sensors_temperatures()`, `sensors_battery()`, and parts of `net_connections()` are the usual portability gaps. The docs mark availability per function, and code targeting multiple OSes must guard for `AttributeError` / `NotImplementedError`.

## Production Notes

- **It is a compiled dependency.** On platforms without a prebuilt wheel (older distros, uncommon architectures, some Alpine/musl or free-threaded setups) `pip install psutil` falls back to building from source and fails without a C compiler and Python headers (`gcc`, `python3-dev`). This is the single most common install failure. In containers, install into an image that has the wheel or the build toolchain.
- **Permissions matter.** `open_files()`, `net_connections()`, `Process.environ()`, and full process enumeration often require elevated privileges. Running unprivileged yields `AccessDenied` for other users' processes; design for partial visibility rather than assuming root.
- **First-sample zeros.** A monitoring loop that reads `cpu_percent()` once per scrape and reports the first value will emit a bogus `0.0`. Keep the same interpreter alive across scrapes so the internal delta is meaningful, or pass an `interval`.
- **`net_connections()` is expensive** on hosts with many sockets; it enumerates the whole connection table each call. Sample it less frequently than cheap counters like `net_io_counters()`.
- **Numbers are OS semantics, not psutil's.** "Memory used" and "available" mean different things across Linux, macOS, and Windows; psutil documents which native field it maps to but does not invent a unified definition. Compare like-for-like fields, and read the docs before alerting on `percent`.
- **Upgrade discipline is good but not zero-cost.** psutil follows the platform tools closely, so field sets on named tuples occasionally gain members between minor releases; unpacking a named tuple by position (rather than by attribute) is the pattern that breaks on upgrade.

## When to Use / When Not

**Use when:**
- You need process/system metrics from Python and want one API across Linux, Windows, macOS, and BSD.
- You're building a monitoring agent, profiler, resource limiter, or process manager and want accurate, maintained OS bindings instead of shelling out to `ps`/`top`.
- You want named-tuple results you can log or serialize directly.

**Avoid when:**
- You need a ready-made metrics endpoint or dashboard, not a library — use an exporter or a monitoring tool instead.
- Your environment can't build or ship a compiled extension and no wheel exists for it.
- You only need one or two numbers on a single OS — the standard library (`os`, `resource`, `platform`) may suffice without a dependency.
- You're not in Python: use a native port rather than embedding an interpreter.

## Alternatives

- shirou/gopsutil — a Go port of psutil's API; use when your agent or service is written in Go.
- nicolargo/glances — a full monitoring TUI/web tool built on top of psutil; use when you want the finished tool, not the library.
- prometheus/node_exporter — use when you want a scrapable Prometheus metrics endpoint rather than in-process Python calls.
- influxdata/telegraf — use when you want a configurable metrics-collection agent shipping to a time-series backend.
- Python stdlib (`os`, `resource`, `platform`) — use when your needs are minimal and single-platform and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2009 | First public release; project originated on Google Code before migrating to GitHub[^1]. |
| 1.0.0 | 2013 | API considered stable. |
| 2.0.0 | 2014-03 | Large breaking redesign: dropped `get_`/`set_` prefixes, reworked process method names[^3]. |
| 3.0.0 | 2015 | Continued API cleanup and new metrics. |
| 5.0.0 | 2016 | Long-lived 5.x line; broad platform and sensor coverage. |
| 6.0.0 | 2024 | Removal of long-deprecated aliases; retired some legacy runtime support[^4]. |
| 7.0.0 | 2025 | Further cleanup on the current major line[^4]. |

Dates for the 5.x/6.x/7.x majors are approximate to the year unless footnoted; consult the changelog for exact releases.

## References

[^1]: psutil README and documentation — project description, supported platforms, and history. https://psutil.readthedocs.io/
[^2]: psutil adoption page — download volume and dependent-repository counts (figures move over time). https://psutil.readthedocs.io/latest/adoption.html
[^3]: psutil 2.0 porting notes — the `get_`/`set_` prefix removal and method renames. https://psutil.readthedocs.io/latest/#psutil-2-0-porting
[^4]: psutil changelog / release history. https://github.com/giampaolo/psutil/blob/master/HISTORY.rst

## Tags

python, system-monitoring, process-management, cross-platform, cpu, memory, disk, network, sensors, observability, c-extension, library
