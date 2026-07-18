# bheisler/criterion.rs

> Statistics-driven microbenchmarking for Rust — regression detection on stable, at the cost of wall-clock noise sensitivity.

[GitHub repo](https://github.com/bheisler/criterion.rs) ·
[Book / docs](https://bheisler.github.io/criterion.rs/book/) ·
[License: Apache-2.0 OR MIT](https://github.com/bheisler/criterion.rs/blob/master/LICENSE-APACHE)

## Overview

Criterion.rs is the de facto standard benchmarking library for Rust. Modeled on Bryan O'Sullivan's Haskell Criterion, it exists because the language's built-in `#[bench]` harness (libtest) has been nightly-only for a decade. Criterion runs on stable Rust and replaces libtest's single-number output with a statistical pipeline: warm-up, sampling, linear regression, bootstrap confidence intervals, and — its defining feature — automatic comparison against the previous run with a significance test ("change: [-2.1% +0.8%] (p = 0.32)"). First published to crates.io in December 2017 by Brook Heisler[^1]; the repository itself dates to 2014, from Jorge Aparicio's original prototype that Heisler took over and rewrote[^2].

The project's central tension is measurement fidelity versus convenience. It measures wall-clock time, which is easy and portable but noisy: CPU frequency scaling, thermal throttling, and shared CI runners routinely produce ±5% swings that its p-value machinery will flag as "regressions." Instruction-count-based tools (iai-callgrind) trade portability for determinism; Criterion chose the opposite corner, and most of its production footguns follow from that choice.

Maintenance history matters here. After 0.5.1 (May 2023) the repo went quiet for two years; 0.6.0 (May 2025) and 0.7.0 (July 2025) shipped under new maintainers, and in 2026 active development moved to a separate `criterion-rs` GitHub organization because Heisler had been absent for extended periods[^3]. The bheisler repo (5.5k stars) still holds the history and most inbound links, but new issues and PRs are directed to criterion-rs/criterion.rs. Repo activity metrics for the old location understate the project's actual liveness.

## Getting Started

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.7"

[[bench]]
name = "my_benchmark"
harness = false          # required — disables libtest's harness
```

```rust
// benches/my_benchmark.rs
use criterion::{criterion_group, criterion_main, Criterion};
use std::hint::black_box;

fn fibonacci(n: u64) -> u64 {
    match n { 0 => 1, 1 => 1, n => fibonacci(n - 1) + fibonacci(n - 2) }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

Run with `cargo bench`; filter with `cargo bench -- fib`. Results and HTML reports land in `target/criterion/`.

## Architecture / How It Works

Each benchmark goes through a fixed pipeline[^4]:

1. **Warm-up** (default 3 s) — runs the routine to stabilize caches, branch predictors, and frequency governors, and to estimate iteration cost.
2. **Measurement** (default 5 s, 100 samples) — samples are taken at linearly increasing iteration counts. Timing many iterations per sample amortizes clock granularity, which is what lets Criterion resolve sub-nanosecond routines with a nanosecond-resolution clock.
3. **Analysis** — a linear regression of total time against iteration count estimates per-iteration time; bootstrap resampling produces confidence intervals for mean, median, slope, and standard deviation. Outliers are classified with Tukey's fences but not discarded.
4. **Comparison** — statistics are compared against the saved run in `target/criterion/` (or a named `--save-baseline`/`--baseline` snapshot), with a significance test and a configurable `noise_threshold` deciding whether to report improvement/regression.

Because `harness = false` hands `main` to Criterion, each bench target is its own binary with Criterion's own CLI (filters, `--quick`, baseline flags). Measurement is pluggable via the `Measurement` trait — third-party crates swap wall time for hardware perf counters (criterion-perf-events) or cycle counts. Async routines are supported through executor adapters for Tokio, async-std, and smol. Plot generation uses gnuplot if found on `$PATH`, silently falling back to the pure-Rust plotters backend; the companion `cargo-criterion` binary offloads report generation out of the library. `black_box` — needed to stop LLVM from constant-folding your benchmark away — is now the std one (`std::hint::black_box`), after years of a volatile-read workaround on stable.

## Production Notes

- **Baselines are machine-local.** Numbers in `target/criterion/` are not portable across machines, CPU generations, or even power profiles. Comparing a laptop baseline against a CI run is meaningless.
- **CI is hostile territory.** Shared runners (GitHub Actions included) have noisy neighbors and variable clocks; expect false regressions at the default thresholds. Common mitigations: raise `noise_threshold`, use dedicated bare-metal runners, or move CI regression-gating to iai-callgrind and keep Criterion for local investigation.
- **Setup leaks into measurement if you let it.** `b.iter(|| ...)` times the whole closure. Per-iteration setup must go through `iter_batched` / `iter_with_setup`, and the `BatchSize` choice (per-iteration vs large-batch) materially changes results for allocation-heavy setup.
- **Missing `harness = false` is the classic first-run failure** — libtest intercepts the bench binary's CLI and Criterion's flags error out.
- **Compile-time cost is real.** Criterion pulls a deep dev-dependency tree (clap, serde, plotters, rayon, …) and every bench target is a separate binary; cold `cargo bench` builds on medium crates take minutes. Disabling default features trims some of it.
- **Wall-time only, by default.** No allocation counts, no cache stats. Reach for the `Measurement` trait, iai-callgrind, or divan's allocation profiling when "faster" needs a why.
- **Watch the repo split.** Pinning `criterion = "0.5"` (still extremely common in the ecosystem) means missing two years of fixes; upstream is now the criterion-rs org[^3], and future releases will come from there.

## When to Use / When Not

**Use when:**
- You need benchmarks on stable Rust with statistical regression detection.
- You are optimizing locally and want HTML reports, historical comparison, and outlier-aware statistics rather than a single mean.
- You benchmark async code or need throughput (bytes/elements per second) reporting.

**Avoid when:**
- You need deterministic CI gating — instruction-count tools (iai-callgrind) don't flap on noisy runners.
- Benchmark-suite compile time or dependency footprint matters — divan is far lighter and its attribute API is terser.
- You are timing whole programs or CLIs — hyperfine operates at process level.
- You need cross-machine comparability — no wall-clock harness gives you that.

## Alternatives

- nvzqz/divan — attribute-macro benchmarks with much lighter dependencies and built-in allocation profiling; use it when Criterion's compile cost and ceremony outweigh its statistics.
- iai-callgrind/iai-callgrind — Cachegrind-based instruction counting; use it for deterministic regression gates in CI (Linux + Valgrind required).
- sharkdp/hyperfine — CLI-level benchmarking of whole commands, not functions.
- bheisler/iai — Heisler's original instruction-count experiment; effectively superseded by iai-callgrind.
- rust-lang libtest `#[bench]` — built-in but nightly-only and statistically naive; only sensible inside the rust-lang toolchain itself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017-12 | First crates.io release under Heisler; stable-Rust harness, gnuplot reports[^1]. |
| 0.2.0 | 2018-02 | Early API expansion; start of the long 0.2.x series. |
| 0.3.0 | 2019-08 | `Measurement` trait for pluggable measurements; plotters backend and async executor support followed in the 0.3.x series. |
| 0.4.0 | 2022-09 | Breaking cleanup after three years of 0.3.x. |
| 0.5.1 | 2023-05 | Last release of the Heisler era. |
| 0.6.0 | 2025-05 | First release after a two-year hiatus, under new maintainers. |
| 0.7.0 | 2025-07 | Current major line. |
| — | 2026 | Development moves to the criterion-rs GitHub organization[^3]. |

## References

[^1]: criterion crate versions on crates.io. https://crates.io/crates/criterion/versions
[^2]: Criterion.rs book, FAQ — project origins and comparison with libtest. https://bheisler.github.io/criterion.rs/book/faq.html
[^3]: bheisler/criterion.rs README — migration notice to the criterion-rs organization. https://github.com/bheisler/criterion.rs#readme
[^4]: Criterion.rs book, "Analysis Process". https://bheisler.github.io/criterion.rs/book/analysis.html

## Tags

rust, benchmarking, performance, statistics, microbenchmark, testing-tools, developer-tools, regression-detection, cargo
