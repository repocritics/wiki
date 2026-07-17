# martinus/nanobench

> Single-header C++ microbenchmarking library that reads Linux perf counters and reports median-with-error in a few tens of milliseconds.

[GitHub repo](https://github.com/martinus/nanobench) ·
[Official website](https://nanobench.ankerl.com) ·
[License: MIT](https://github.com/martinus/nanobench/blob/master/LICENSE)

## Overview

`ankerl::nanobench` is a microbenchmarking library for C++11/14/17/20 by Martin Ankerl (martinus). It ships as a single header (`nanobench.h`) with the implementation compiled into exactly one translation unit via a `#define` — the same include-once pattern used by stb-style libraries. The design target is a benchmark that answers "which of these two snippets is faster" quickly and defensibly, not a full statistical suite.

Its distinguishing feature is that on Linux it reads hardware performance counters through the `perf_event_open` syscall and reports not just wall-clock ns/op and op/s but instructions per operation, CPU cycles, IPC, branches, and branch-miss rate. That extra signal is what lets you reason about *why* one variant is faster — fewer instructions, better branch prediction, or fewer stalls — rather than only *that* it is. The project advertises being roughly 80× faster to produce a result than Google Benchmark[^1], achieved by running short epochs and reporting a median with a median-absolute-error percentage instead of grinding a fixed long wall-clock budget.

The honest tension: nanobench trades ceremony and cross-platform parity for speed and ergonomics. The perf-counter columns only exist on Linux; on macOS and Windows you get timing only. And a "fast, single-header" benchmark makes it easy to write a *fast wrong* benchmark — the dead-code-elimination and frequency-scaling footguns that plague all microbenchmarks are still yours to manage.

## Getting Started

There is no build step — vendor the single header, or install via a package manager (Conan, vcpkg, and Meson wrap all carry it). Define the implementation macro in exactly one `.cpp`:

```cpp
#define ANKERL_NANOBENCH_IMPLEMENT
#include <nanobench.h>

int main() {
    double d = 1.0;
    ankerl::nanobench::Bench().run("some double ops", [&] {
        d += 1.0 / d;
        if (d > 5.0) { d -= 5.0; }
        ankerl::nanobench::doNotOptimizeAway(d);
    });
}
```

Every other translation unit just `#include <nanobench.h>` without the macro. `doNotOptimizeAway` is the load-bearing call: without it the optimizer deletes the loop body and you benchmark nothing.

## Architecture / How It Works

A `Bench` object holds configuration (name, iteration counts, output target) and `run(name, lambda)` executes the measured closure. Internally nanobench runs the closure in **epochs** — several independent measurement batches (11 by default) — auto-sizing the iteration count per epoch to hit a target measurement time, then reports the **median** across epochs plus an error percentage derived from the median absolute deviation. Median-of-epochs is deliberately robust against outliers (a scheduler hiccup skews one epoch, not the reported number), which is why nanobench can converge in tens of milliseconds where fixed-budget tools run for seconds.

On Linux the measurement wraps each epoch in `perf_event_open` counters for instructions, cycles, branch instructions, and branch misses. These are exact hardware counts, not sampled estimates, which is what makes the `ins/op` and `IPC` columns trustworthy when they are present. When the syscall is unavailable (no permission, non-Linux, containerized without the capability), nanobench silently drops those columns and reports timing only.

Output is driven by a small mustache-style template engine. Built-in renderers produce the default markdown table, CSV, JSON, a `pyperf`-compatible format, and an HTML box-plot page you can drop into a browser to visualize distribution across runs. Comparative benchmarking uses `.relative(true)`: the first benchmark in a group is treated as the 100% baseline and subsequent rows report as a percentage of it. A small fast PRNG (`ankerl::nanobench::Rng`) is bundled so benchmarks can generate inputs without pulling in `<random>`'s overhead.

## Production Notes

- **perf counters need permission.** `perf_event_open` is gated by `/proc/sys/kernel/perf_event_paranoid`. On a default hardened host, CI runner, or Docker container without `CAP_PERFMON`/`CAP_SYS_ADMIN`, the counter columns vanish with no error — you get timing-only output and may not notice you lost the signal you came for. Set `perf_event_paranoid` to 1 or lower (or grant the capability) if you need `ins/op`.
- **Frequency scaling wrecks accuracy.** Turbo boost and CPU frequency governors make the same code measure differently run-to-run. nanobench warns when it detects unreliable conditions, but it cannot fix them — pin the governor to `performance` and disable turbo for stable numbers. This is the single most common cause of "my err% is 10%" reports.
- **Dead-code elimination is your problem.** `doNotOptimizeAway` on both inputs and outputs is mandatory; forgetting it silently benchmarks an empty loop. A result that looks impossibly fast (sub-nanosecond for real work) almost always means the optimizer ate the body.
- **Cross-platform asymmetry.** Treat perf-counter columns as Linux-only. A benchmark suite that asserts on `ins/op` will behave differently in CI depending on OS and container privileges.
- **Not a load/throughput harness.** nanobench measures tight in-process code paths. It has no fixtures, no multi-threaded contention model, no statistical significance testing across configurations — it is a stopwatch with good counters, not a benchmarking framework in the JMH sense.
- **Maintenance cadence is quiet.** The last push was October 2024 and the library is mature and stable rather than fast-moving. For a header-only benchmarking tool this is largely fine — the API has been stable across the 4.x line — but do not expect rapid response to issues.

## When to Use / When Not

**Use when:**
- You want to A/B two implementations quickly and see instruction/cycle/branch counts, not just time.
- You value a single-header, zero-build-system dependency you can vendor into any C++ project.
- You benchmark on Linux and want cheap access to hardware perf counters without wiring up `perf` yourself.
- You want machine-readable output (CSV/JSON) or HTML box plots for tracking a number over time.

**Avoid when:**
- You need cross-platform perf-counter parity — macOS/Windows give timing only.
- You need fixtures, parameterized benchmark matrices, or statistical A/B significance testing (Google Benchmark / Celero fit better).
- You are measuring whole-program throughput, concurrency, or I/O rather than tight CPU-bound snippets.
- Your environment forbids relaxing `perf_event_paranoid` and you specifically need the counter columns.

## Alternatives

- google/benchmark — the feature-complete standard: fixtures, arguments, complexity estimation, threading; heavier setup and much slower to a result.
- DigitalInBlue/Celero — richer experiment/fixture model with baselines; use when you want structured benchmark suites over quick snippets.
- catchorg/Catch2 — its `BENCHMARK` macro is convenient when you already use Catch2 for tests and want timing without adding a dependency.
- martinus/robin-hood-hashing — same author; unrelated (a hash map), but the repo nanobench was originally built to benchmark.
- p-ranav/criterion — header-only C++ micro-benchmarking with a statistics focus; alternative when you want confidence-interval reporting.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2019-10-05 | Repository created[^2]. |
| 3.x | 2019–2020 | Early single-header releases; perf-counter integration and markdown output. |
| 4.0.0 | 2020 | Major API cleanup; the `Bench().run()` interface used today[^3]. |
| 4.3.x | 2022–2024 | Incremental fixes; last push October 2024[^2]. |

## References

[^1]: nanobench documentation, "Comparison" — runtime vs. Google Benchmark. https://nanobench.ankerl.com/comparison.html
[^2]: GitHub API metadata for martinus/nanobench (created 2019-10-05, last push 2024-10-06). https://github.com/martinus/nanobench
[^3]: nanobench documentation and reference. https://nanobench.ankerl.com/reference.html

## Tags

cpp, benchmarking, microbenchmark, header-only, single-header, performance, perf-counters, linux, profiling, c-plus-plus
