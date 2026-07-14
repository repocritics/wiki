# sharkdp/hyperfine

> A command-line benchmarking tool that runs a command many times, corrects for shell startup, and reports mean ± standard deviation with statistical outlier warnings.

[GitHub repo](https://github.com/sharkdp/hyperfine) ·
[License: Apache-2.0](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-APACHE) (dual-licensed MIT OR Apache-2.0[^1])

## Overview

hyperfine is a wall-clock benchmarking tool for whole commands. You give it one or more shell commands; it executes each repeatedly, times the runs, and prints summary statistics (mean, standard deviation, min, max, median) plus a relative comparison between commands. It was written by David Peter (`sharkdp`), the same author as `fd`, `bat`, `hexyl`, and `numbat`, and first appeared in early 2018[^2]. It is written in Rust and is inspired by Gabriella Gonzalez's Haskell `bench` tool[^3].

The defining design choice is that hyperfine benchmarks *processes*, not functions. It does not instrument code, sample a profiler, or read hardware counters — it measures the end-to-end time to spawn and complete a command. To make small differences meaningful, it does two things carefully: it runs an adaptive number of iterations (by default at least 10 runs and at least 3 seconds), and it calibrates the startup cost of the intermediate shell by timing the shell with an empty command, then subtracts that from every measurement[^4]. This is what lets it compare, say, `fd` against `find` and report a defensible ratio.

The tension worth understanding up front: hyperfine gives you a clean mean ± stddev and a tidy Markdown table, which is easy to over-trust. Wall-clock benchmarking on a shared, frequency-scaling, thermally-throttling machine is inherently noisy, and the tool's own statistics assume a roughly well-behaved distribution. It surfaces outlier and interference warnings, but it will not stop you from drawing a conclusion from two overlapping distributions. It is a measurement instrument, not a judgment.

## Getting Started

```sh
# Homebrew (macOS), apt (Ubuntu/Debian), dnf, pacman, choco/scoop/winget, ...
brew install hyperfine
# or from source (requires Rust 1.76+)
cargo install --locked hyperfine
```

```sh
# Compare two commands; hyperfine picks the run count automatically
hyperfine 'fd -e jpg' 'find . -iname "*.jpg"'

# Warm the disk cache first, then export a Markdown comparison table
hyperfine --warmup 3 --export-markdown bench.md \
  'grep -R TODO src' 'rg TODO src'

# For very fast commands (<5 ms), skip the shell to avoid startup noise
hyperfine -N './target/release/mytool --version'
```

## Architecture / How It Works

The measurement loop is straightforward but the details are where the value is:

- **Adaptive run count.** Unless you pin it with `-r`/`--runs` (or `--min-runs`/`--max-runs`), hyperfine keeps timing until it has both a minimum number of samples and a minimum wall-clock budget. Cheap commands therefore get many samples; expensive ones get fewer.
- **Shell-spawn correction.** By default each command is run through an intermediate shell (`/bin/sh` on Unix, `cmd.exe` on Windows) so that pipes, globs, and redirection work. hyperfine separately calibrates how long that shell takes to start with an empty command and subtracts it. `-N`/`--shell=none` disables both the shell and the correction, which is the right choice for sub-5 ms commands where the correction's own variance dominates — at the cost of losing shell syntax (`*`, `~`, pipes).
- **Lifecycle hooks.** `--setup` runs once before all runs of a command, `--prepare` runs before *each* timed run (e.g. dropping the page cache to force a cold-cache benchmark), `--cleanup` runs once after. Only the timed command itself is measured; hook time is excluded.
- **Statistics and outliers.** It reports mean, stddev, median, min, max, and a relative factor between commands. It applies a modified-Z-score outlier check and emits warnings when it sees a statistical outlier, when the first run is much slower than the rest (a cache-warming signal, suggesting `--warmup`), or when runtimes look bimodal.
- **Parameterization.** `--parameter-scan name lo hi` (with optional `-D` step size) and `--parameter-list name a,b,c` expand a single benchmark spec into a family of runs, substituting `{name}` into the command. This is the mechanism behind thread-count sweeps and compiler comparisons.
- **Export.** Results serialize to CSV, JSON, Markdown, and AsciiDoc. The JSON carries every individual timing, which is the intended integration point — the repo ships `scripts/` (Python) for histograms and whisker plots, and third-party tools consume the same JSON.

There is no daemon and no persistent state; each invocation is self-contained. Cross-platform support is real (Linux, macOS, Windows, BSDs), though what the numbers *mean* differs by platform because the OS scheduler and timer resolution differ.

## Production Notes

The tool is small and stable; the footguns are about interpretation, not bugs.

- **Wall-clock only.** hyperfine measures elapsed real time, not CPU time, not memory, not syscalls. A command that is faster because it used more cores looks faster; a command starved by a noisy neighbor looks slower. It is a comparison instrument, valid only to the extent the environment is held constant.
- **Environment noise dominates small deltas.** CPU frequency scaling, turbo boost, thermal throttling, and background processes routinely produce single-digit-percent swings. If two commands' `mean ± stddev` ranges overlap, treat them as tied. Pin the governor to `performance` and close other work for anything you'll publish.
- **The shell correction has a noise floor.** For commands in the millisecond range, the subtracted shell-startup estimate carries its own variance and can swamp the signal. Use `-N`. Conversely, `-N` changes *what* you measure (no shell), so don't mix `-N` and shell-mode numbers in one comparison.
- **Cold vs warm cache is a decision, not a default.** I/O-heavy benchmarks are dominated by the page cache. Use `--warmup N` for steady-state warm-cache numbers, or `--prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches'` (Linux) for cold-cache numbers. Reporting one without saying which is a common way to mislead.
- **Not a microbenchmark tool.** There is per-process spawn overhead on every run, so hyperfine cannot resolve sub-millisecond function-level differences. For in-process Rust microbenchmarks use criterion; for C++ use google/benchmark.
- **JIT/GC warmup breaks the normality assumption.** Benchmarking JVM, Node, or Python programs mixes cold-start and warm-steady-state runs, producing heavy-tailed or bimodal distributions that a single mean misrepresents. Increase `--warmup`, or benchmark the steady state explicitly.
- **Upgrades are low-drama.** The CLI surface has been stable across the 1.x line; option flags added over the years (`--shell`, `--setup`, `-N`) are additive. `cargo install` needs a reasonably recent Rust toolchain (1.76+ as of v1.20.0).

## When to Use / When Not

**Use when:**
- You want to compare the end-to-end runtime of two or more commands or CLI tools.
- You need a repeatable, scriptable benchmark with machine-readable (JSON/CSV) output for CI or dashboards.
- You want shell-startup correction, warmup, and cache-control handled for you rather than hand-rolling a `time`-in-a-loop script.
- You're sweeping a parameter (threads, buffer size, compiler) and want a table out of it.

**Avoid when:**
- You need sub-millisecond, function-level microbenchmarks — use an in-process harness (criterion, google/benchmark).
- You need CPU time, allocations, cache misses, or hardware counters — use `perf`, `valgrind`/`callgrind`, or a profiler.
- Your workload's distribution is heavy-tailed or multimodal (JIT/GC) and a single mean would mislead more than inform.
- You cannot control the environment (shared CI runners with unknown neighbors) and need publication-grade precision.

## Alternatives

- bheisler/criterion.rs — in-process statistical microbenchmarking for Rust; use it when you're timing functions inside one process, not whole commands.
- google/benchmark — C++ microbenchmark library with the same in-process focus; use it for C/C++ hot loops.
- Gabriella439/bench — the Haskell CLI benchmarker that inspired hyperfine; use it if you're already in a Haskell/Nix workflow.
- ionelmc/pytest-benchmark — pytest fixture for benchmarking Python functions; use it when the thing under test is Python code, not a command.
- sharkdp/hyperfine — (this) when the unit of comparison is a full command invocation and wall-clock is the right metric.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-01 | Initial public release; repo created 2018-01-13[^2]. |
| 1.11.0 | 2020-10 | Ongoing 1.x maturation of export formats and options[^5]. |
| 1.15.0 | 2022-09 | Continued option/export refinements[^5]. |
| 1.18.0 | 2023-10 | `--shell`/`--shell=none` (`-N`) intermediate-shell control matured[^5]. |
| 1.19.0 | 2024-11 | Maintenance and analysis-script updates[^5]. |
| 1.20.0 | 2025-11 | Latest release; requires Rust 1.76+[^5]. |

## References

[^1]: hyperfine README, "License" — dual-licensed under MIT and Apache-2.0. GitHub's repository metadata reports the SPDX id as `Apache-2.0`. https://github.com/sharkdp/hyperfine#license
[^2]: hyperfine repository — created 2018-01-13. https://github.com/sharkdp/hyperfine
[^3]: hyperfine README, "Alternative tools" — inspired by `bench`. https://github.com/Gabriella439/bench
[^4]: hyperfine README, "Intermediate shell" — shell-spawn calibration and `--shell=none`. https://github.com/sharkdp/hyperfine#intermediate-shell
[^5]: hyperfine releases. https://github.com/sharkdp/hyperfine/releases

## Tags

rust, cli, benchmarking, command-line, performance, statistics, developer-tools, cross-platform, terminal
