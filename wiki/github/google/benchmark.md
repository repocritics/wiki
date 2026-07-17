# google/benchmark

> A C++ microbenchmark library with a GoogleTest-style API — register a function, loop over `state`, and get statistically-aggregated timing.

[GitHub repo](https://github.com/google/benchmark) ·
[License: Apache-2.0](https://github.com/google/benchmark/blob/main/LICENSE)

## Overview

Google Benchmark (the `benchmark` library, often "gbench") is a C++ library for
measuring the wall-clock and CPU cost of small code snippets[^1]. It occupies the
same niche for performance measurement that GoogleTest occupies for correctness
testing, and deliberately mirrors its structure: you write a function, register
it with a macro, and a generated `main` runs and reports every registered case.
The project has been public since 2013 and is maintained by Google engineers as
the de facto standard microbenchmark harness in the C++ ecosystem[^2].

The defining tension of the library is that microbenchmarking is *adversarial to
the optimizer*. An optimizing C++ compiler exists to delete work whose result is
unused — exactly the work a benchmark tries to measure. Google Benchmark does not
solve this for you; it gives you the tools (`DoNotOptimize`, `ClobberMemory`, a
`state` loop that resists hoisting) and expects you to apply them correctly. A
benchmark that forgets them will happily report that an expensive function takes
zero nanoseconds — the single most common way to get wrong numbers out, and a
property of the problem, not a bug.

It is a build-time-heavy dependency: usable from C++11 code but requiring a C++17
compiler and standard library to *build* itself[^3], with a test suite that
depends on GoogleTest. Most projects pull it in only for their `bench/` target and
disable its tests and install rules.

## Getting Started

The library builds with CMake or Bazel. With CMake and vendored dependencies:

```bash
git clone https://github.com/google/benchmark.git
cd benchmark
cmake -E make_directory build
cmake -DBENCHMARK_DOWNLOAD_DEPENDENCIES=on -DCMAKE_BUILD_TYPE=Release -S . -B build
cmake --build build --config Release
```

A minimal benchmark. Note `DoNotOptimize` — without it the compiler may delete
the string construction entirely:

```c++
#include <benchmark/benchmark.h>
#include <string>

static void BM_StringCopy(benchmark::State& state) {
  std::string x = "hello world";
  for (auto _ : state) {          // only this body is timed
    std::string copy(x);
    benchmark::DoNotOptimize(copy); // force the result to be "used"
  }
}
BENCHMARK(BM_StringCopy);

BENCHMARK_MAIN();                 // generates main(); runs all registered cases
```

```bash
g++ mybench.cc -std=c++17 -isystem benchmark/include \
  -Lbenchmark/build/src -lbenchmark -lpthread -o mybench
./mybench --benchmark_repetitions=10 --benchmark_report_aggregates_only=true
```

## Architecture / How It Works

**Static registration.** `BENCHMARK(fn)` expands to a static object whose
constructor registers `fn` in a global registry before `main` runs. This is why
benchmark registrations can silently vanish: if the registrations live in a
`STATIC` library and nothing references them, the linker is free to drop the
translation unit. The documented fix is to place benchmarks in an `OBJECT`
library or the final executable, or use `--whole-archive`[^4].

**The `state` loop.** `for (auto _ : state)` is the timed region. The library
starts the clock on entry, stops on exit, and chooses the iteration count
adaptively — running the body until total wall time crosses a threshold
(`--benchmark_min_time`, default 0.5s), so a sub-nanosecond body still accumulates
a stable sample. You get per-iteration time, not per-call time.

**Optimizer barriers.** `DoNotOptimize(x)` emits an inline-asm constraint marking
`x` as read (and possibly clobbered), preventing dead-code elimination of the
computation behind it. `ClobberMemory()` is a full compiler memory barrier, used
after writes to force stores to be observable. These are the library's
load-bearing primitives.

**Parameterization and timing.** Benchmarks can be swept over arguments (`->Arg`,
`->Range`, `->RangeMultiplier`), run as templates over types
(`BENCHMARK_TEMPLATE`), grouped into fixtures (`BENCHMARK_F`), run across thread
counts (`->Threads`), and asked to infer asymptotic complexity (`->Complexity()`
reports an estimated BigO and RMS fit). Timing reports both CPU and real
(wall-clock) time by default; `->UseRealTime()` suits blocking work, and
`->UseManualTime()` + `state.SetIterationTime()` handle regions the framework
cannot bracket. `state.counters[...]`, `SetBytesProcessed`, and
`SetItemsProcessed` add throughput columns.

**Statistics and output.** `--benchmark_repetitions=N` produces mean, median,
stddev, and coefficient of variation. Output formats are console, JSON, and CSV
(deprecated). `tools/compare.py` diffs two JSON runs and runs a U-test to flag
whether a change is statistically significant[^5] — the intended way to answer
"did my patch help", not eyeballing two console dumps.

## Production Notes

**Build in Release.** By default the library builds as a *debug* library and
prints a runtime warning saying so. Debug numbers are meaningless for the code
under test; always configure `-DCMAKE_BUILD_TYPE=Release`, and build the code
being measured with optimizations too.

**CPU frequency scaling wrecks reproducibility.** The runner warns when it detects
scaling governors enabled. Turbo boost, thermal throttling, and
`ondemand`/`schedutil` governors make consecutive runs disagree by double digits.
Serious measurement requires pinning the governor to `performance`, disabling
turbo, and ideally pinning to an isolated core — the library only warns, it cannot
do this for you.

**Numbers are not portable across machines.** Absolute nanoseconds are meaningful
only relative to other benchmarks on the same silicon, kernel, and compiler. Do
not commit "X takes 40ns" as a cross-CI assertion; compare deltas on one host with
`compare.py` instead.

**`DoNotOptimize` is easy to misapply.** Forgetting it, or passing a value the
compiler can still prove dead, yields impossibly fast results — a 0ns report or a
flat line across `Range` sizes is the classic symptom of an optimized-away body.

**Missing registrations.** Per the static-registration story above, benchmarks in
an intermediate static library often just don't run, with no error. Use an object
library for shared benchmark sources.

**Header layout is changing.** The stable `main` branch uses the monolithic
`<benchmark/benchmark.h>`. An experimental `v2` branch splits the API into smaller
headers and offers no stability guarantee — do not build production tooling
against `v2`[^6].

## When to Use / When Not

**Use when:**
- You need repeatable, statistically-aggregated timing of small C++ functions.
- You want sweeps over input size, thread count, or types with complexity fitting.
- You need machine-readable JSON output and A/B regression comparison in CI.
- Your team already uses GoogleTest and wants a familiar registration model.

**Avoid when:**
- You need whole-program or production profiling — reach for `perf`, VTune, or a
  sampling profiler; microbenchmarks measure snippets, not systems.
- You want a zero-config, single-header drop-in with no build-system surgery —
  the CMake/Bazel + C++17-to-build requirement is heavy for a quick check.
- You are benchmarking allocation-heavy or I/O-bound code where microbench
  isolation hides the real-world cost.
- You are not in C++ (use a language-native harness).

## Alternatives

- martinus/nanobench — single-header, near-zero build cost, prints markdown/JSON; use when you want quick numbers without a CMake dependency.
- catchorg/Catch2 — has a built-in `BENCHMARK` facility; use when you already run Catch2 tests and want benchmarking in the same binary.
- google/googletest — the sibling correctness-testing framework, not a benchmark tool; use for unit tests, not timing.
- bheisler/criterion.rs — the closest equivalent in the Rust ecosystem (statistics-first, HTML reports); use when your code is Rust.
- ableton/blt or hand-rolled `std::chrono` loops — use only for throwaway one-off measurements where statistical rigor does not matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-12 | Repository opened; early internal Google microbenchmark tool[^2]. |
| 1.0.0 | 2016 | First tagged stable release; `BENCHMARK` / `State` API. |
| 1.5.x | 2020 | Complexity reporting, per-benchmark counters matured. |
| 1.6.0 | 2021 | Build/tooling modernization; C++ standard requirements tightened. |
| 1.7.x | 2022 | Coefficient-of-variation aggregate, output refinements. |
| 1.8.x | 2023 | C++17 required to build the library itself[^3]. |
| 1.9.x | 2024–2025 | Ongoing stable line on `main`; `v2` opened for experimental API[^6]. |

## References

[^1]: Google Benchmark README — "A library to benchmark code snippets, similar to unit tests." https://github.com/google/benchmark
[^2]: `google/benchmark` repository metadata (created 2013-12-12), fetched via GitHub API 2026-07. https://github.com/google/benchmark
[^3]: Google Benchmark README, Requirements — "can be used with C++11. However, it requires C++17 to build." https://github.com/google/benchmark#requirements
[^4]: Google Benchmark README, Usage with CMake — note on static-registration symbols and object libraries. https://github.com/google/benchmark#usage-with-cmake
[^5]: Google Benchmark tools documentation — `compare.py`. https://github.com/google/benchmark/blob/main/docs/tools.md
[^6]: Google Benchmark README, Stable and Experimental Library Versions — the `v2` branch. https://github.com/google/benchmark/tree/v2

## Tags

cpp, benchmarking, performance, microbenchmark, profiling, testing, google, cmake, bazel, library
