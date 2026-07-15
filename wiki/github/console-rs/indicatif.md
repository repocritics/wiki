# console-rs/indicatif

> Terminal progress bars and spinners for Rust — the de facto standard for CLI progress reporting.

[GitHub repo](https://github.com/console-rs/indicatif) ·
[Docs (docs.rs)](https://docs.rs/indicatif/) ·
[License: MIT](https://github.com/console-rs/indicatif/blob/main/LICENSE)

## Overview

indicatif is a Rust library for indicating progress to users of command line
applications. It provides progress bars, spinners, multi-bar layouts, and basic
styled/colored output. Originally written by Armin Ronacher and now maintained
under the `console-rs` organization, it is the crate most Rust CLIs reach for
when they need a progress bar; downloads, build tools, and installers across the
ecosystem depend on it[^1].

The library is deliberately narrow in scope. It does one job — draw progress
state to a terminal without corrupting the surrounding output — and leans on its
sibling crate `console` for the low-level terminal handling (TTY detection, ANSI
styling, cursor movement)[^2]. It is not a TUI framework: there is no layout
engine, event loop, or widget tree. If you need interactive panels, that is
ratatui's territory, not indicatif's.

The defining tension is between simplicity of the happy path and the number of
terminal-behavior footguns behind it. A single `ProgressBar::new(n)` with
`inc(1)` in a loop is three lines. But the moment you add concurrent bars,
interleaved `println!`, non-TTY output (CI logs, pipes), or a spinner that must
animate while your thread is blocked, you hit the parts of the API that exist
specifically to work around how terminals actually behave. Most production
complaints trace back to not knowing which of those escape hatches to use.

## Getting Started

```bash
cargo add indicatif
```

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn main() {
    let pb = ProgressBar::new(1024);
    pb.set_style(
        ProgressStyle::with_template(
            "{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] \
             {bytes}/{total_bytes} ({eta})",
        )
        .unwrap()               // template parse is fallible — see Production Notes
        .progress_chars("#>-"),
    );

    for _ in 0..1024 {
        pb.inc(1);
        std::thread::sleep(Duration::from_millis(2));
    }
    pb.finish_with_message("done");
}
```

`ProgressStyle::with_template` parses a template string with named placeholders
(`{bar}`, `{pos}`, `{len}`, `{eta}`, `{bytes}`, `{msg}`, `{spinner}`, …). The
inline `:40.cyan/blue` syntax sets width and fill/background colors.

## Architecture / How It Works

A `ProgressBar` is a thin handle around shared interior state (an `Arc` over a
mutex-guarded struct). It is `Send + Sync` and cheap to `clone`; every clone
points at the same underlying bar, which is how you share one bar across threads
or hand it to a closure[^3].

Three pieces do the real work:

- **`ProgressState`** — the numbers: position, length, elapsed time, message,
  and the moving-average estimator that produces `eta`/`per_sec`.
- **`ProgressStyle`** — a compiled template plus the character sets for the bar
  and spinner. Parsing happens once at `with_template`; rendering substitutes
  live state on each draw.
- **`ProgressDrawTarget`** — where and how often output is written. This is the
  most important and least understood component. The draw target rate-limits
  redraws (roughly a capped refresh frequency, not one redraw per `inc`), so
  calling `inc(1)` a million times is cheap — most calls only update the counter
  and skip drawing. On a non-interactive stream (piped stdout, CI), the default
  draw target is **hidden**: the bar updates its numbers but writes nothing.

**`MultiProgress`** coordinates several bars sharing one terminal region. It
owns the draw target and re-renders the whole block on change so bars don't
overwrite each other. Bars are added with `add`/`insert`; finished bars stay on
screen unless cleared.

**Spinner animation** is the subtle part. A bar only redraws when you call it.
If your thread blocks (network, disk) the spinner freezes. `enable_steady_tick`
solves this by spawning a background thread that ticks the bar on an interval —
so an idle spinner keeps spinning — at the cost of a thread per steady-ticked
bar.

There are ergonomic wrappers: `wrap_iter`/`with_iter` to drive a bar from an
iterator's length, and `wrap_read`/`wrap_write` to attach byte progress to an
`io::Read`/`Write`. A `rayon` feature adds parallel-iterator integration.

## Production Notes

- **`println!` while a bar is live corrupts the display.** The bar owns cursor
  lines; a raw `print!`/`println!` interleaves with its redraws and leaves
  garbage. Use `pb.println("…")` (prints above the bar) or `pb.suspend(|| …)`
  (clears, runs your closure, redraws). For logging, route through
  `indicatif-log-bridge` (log crate) or `tracing-indicatif` (tracing) rather
  than fighting the terminal[^4][^5].
- **Non-TTY output is silent by default.** In CI, under `tee`, or when stdout is
  piped, the default hidden draw target produces no progress output at all — a
  frequent "it works locally, nothing in CI" surprise. Force output by
  constructing an explicit `ProgressDrawTarget` if you need progress in logs
  (accepting that it will emit many redraw lines).
- **Templates are fallible.** `with_template` returns a `Result`; an
  unrecognized placeholder or malformed template errors there. Unwrapping a
  bad template panics at startup. This changed in 0.17, which reworked the
  template engine — code written against the pre-0.17 `.template()` API needs
  migration[^6].
- **ETA is a moving-average estimate.** It works well for uniform work and is
  jumpy for bursty or front/back-loaded workloads. Do not treat it as a
  guarantee.
- **Draw throttling means you cannot force-render every tick cheaply.** This is
  usually what you want, but if you rely on a specific frame being drawn you may
  need `tick()` explicitly or an adjusted refresh rate.
- **Steady tick spends a thread.** Enabling it on many concurrent bars spawns
  many ticker threads; prefer a single `MultiProgress` with a shared cadence for
  large fan-outs.
- **Unicode width matters.** Wide CJK characters and emoji in messages can
  miscount display width and cause the line to wrap or smear on redraw; keep
  dynamic messages ASCII-ish where you can.
- **`finish` vs `finish_and_clear`.** `finish` leaves the completed bar on
  screen; `finish_and_clear` removes it. Under `MultiProgress`, un-cleared
  finished bars accumulate.

## When to Use / When Not

**Use when:**
- You have a CLI with a countable or byte-measured task and want a progress bar
  or spinner with minimal ceremony.
- You need multiple concurrent bars (parallel downloads, multi-stage builds)
  coordinated in one terminal region.
- You want progress that plays nicely with `log`/`tracing` via the bridge crates.

**Avoid when:**
- You are building an interactive full-screen terminal UI with panels and input
  — use ratatui; indicatif is output-only.
- Your program's output is primarily machine-consumed logs — progress bars are
  hidden or noisy there; structured logging is a better fit.
- You only need colored text and no progress — depend on `console` (or `owo-colors`)
  directly and skip the bar machinery.

## Alternatives

- console-rs/console — the lower-level terminal styling/TTY crate indicatif is
  built on; use it directly when you need colors and terminal control but no bars.
- ratatui-org/ratatui — full TUI framework with a `Gauge` widget; use when
  progress is one part of an interactive full-screen interface.
- a8m/pb — older `pbr` crate with a simpler single-bar API; use when you want a
  minimal bar without the template system.
- fosskers/linya — minimal, low-dependency concurrent progress bars; use when
  binary size / dependency count matters more than features.
- FGRibreau/spinners — spinner animations only; use when you need a loading
  spinner and nothing measurable to report.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-04-23 | Repository created; progress bars + spinners on top of `console`[^1]. |
| 0.15.0 | 2020-06-14 | Iterator/read/write wrappers, styling refinements. |
| 0.16.0 | 2021-04-30 | API and rendering improvements. |
| 0.17.0 | 2022-07-30 | Template engine rework; `with_template` returns `Result`[^6]. |
| 0.18.0 | 2025-07-04 | New minor line after a ~3-year 0.17 series. |
| 0.18.6 | 2026 | Current release line at time of writing. |

## References

[^1]: indicatif README and repository. https://github.com/console-rs/indicatif
[^2]: `console` — terminal abstraction crate used by indicatif. https://github.com/console-rs/console
[^3]: `ProgressBar` API documentation. https://docs.rs/indicatif/latest/indicatif/struct.ProgressBar.html
[^4]: `indicatif-log-bridge` — integrate with the `log` crate. https://crates.io/crates/indicatif-log-bridge
[^5]: `tracing-indicatif` — integrate with the `tracing` crate. https://crates.io/crates/tracing-indicatif
[^6]: `ProgressStyle` / template documentation. https://docs.rs/indicatif/latest/indicatif/struct.ProgressStyle.html

## Tags

rust, cli, progress-bar, spinner, terminal, tui, console, command-line, cargo-crate, developer-tools
