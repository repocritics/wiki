# rsalmei/alive-progress

> A single-loop terminal progress bar for Python with a throughput-reactive spinner, exponential-smoothing ETA, and stdout/logging hooks that keep printed output above the animation.

[GitHub repo](https://github.com/rsalmei/alive-progress) ·
[PyPI](https://pypi.org/project/alive-progress/) ·
[License: MIT](https://github.com/rsalmei/alive-progress/blob/main/LICENSE)

## Overview

`alive-progress` is a Python progress-bar library authored by Rogério Sampaio de Almeida (`rsalmei`), first published in 2019[^1]. Its distinguishing idea is that the animation is driven by a background daemon thread that refreshes the terminal at a bounded frame rate, decoupled from how fast your loop actually runs. Your code only calls `bar()` to increment a counter; the display speed reflects observed throughput rather than redrawing on every iteration. At roughly a million iterations per second this collapses to about 60 screen updates per second, which keeps CPU cost and terminal spam low.

The library leans into presentation. It ships a large catalog of spinners and bar styles, a spinner "compiler" that renders animation frames ahead of time, an ETA computed with exponential smoothing, and a printed final receipt with totals, elapsed time, and throughput. It also owns two features most competitors lack: an interactive pause/resume mechanism that drops you back to the REPL mid-run, and grapheme-aware rendering so emoji and wide Unicode characters align correctly inside bars and spinners.

The defining tradeoff is scope. `alive-progress` is deliberately a single-bar tool. It does not support nested or multiple concurrent bars, and it takes over `sys.stdout` while active. That makes it excellent for one long-running loop in a script or CLI and a poor fit when the progress indicator must coexist with a larger terminal UI or several parallel tasks — which is exactly where `tqdm` and `rich` are stronger.

## Getting Started

```bash
pip install alive-progress
```

```python
from alive_progress import alive_bar
import time

with alive_bar(1000) as bar:           # known total -> auto mode with ETA
    for item in range(1000):
        time.sleep(0.005)              # your work here
        bar()                          # advance one step
```

```python
# Iterator adapter — calls bar() for you
from alive_progress import alive_it

for item in alive_it(range(1000), title="Processing"):
    ...  # do work; the bar advances automatically
```

## Architecture / How It Works

The core is a **decoupled refresh loop**. `alive_bar()` is a context manager that spawns a background thread rendering the current widget row at a controlled FPS. The driving loop mutates lightweight state — the counter, `bar.text`, `bar.title` — and the renderer reads it. Because rendering frequency is independent of iteration frequency, a tight loop does not translate into a redraw per step; the bar samples state instead.

Rendering is built on what the author calls **cell architecture**, introduced in the 2.0 rewrite. Instead of treating output as characters, every component (spinner, bar, tip, border, text) produces streams of *cells*. Grapheme clusters — emoji and other wide characters that occupy one visual position but multiple code points — are handled as single units that occupy two cells, so animations stay aligned even when a wide glyph is partially scrolled off an edge. Grapheme segmentation is provided by the `graphemeu` dependency, a maintained fork adopted in the 3.3 series after the original `grapheme` package went unmaintained.

Spinners are **compiled ahead of time**. A spinner factory is expanded into all frames of all animation cycles once, up front, and playback is then just indexing into precomputed cells with essentially no per-frame overhead. The `check()` tooling exposes this by exploding every frame and cycle on screen, which is how custom spinners and bars are meant to be designed and debugged.

The **print/logging hooks** are the other major mechanism. While a bar is active the library redirects `sys.stdout` (and installs a logging hook) so that `print()` and log records are flushed cleanly above the animated row rather than corrupting it, and are optionally enriched with the bar's position (`on 42:`). As of the 3.2 series these hooks are synchronized for multithreaded callers, so multiple threads can print without a message queue. ETA and rate are both smoothed with an exponential-smoothing estimator, which is why the ETA stays stable instead of jumping on every sampled interval.

## Production Notes

- **It replaces `sys.stdout` while active.** This is the most common source of surprises. Any library that also captures or reconfigures stdout, does its own cursor control, or expects a stable stream object can conflict with an active `alive_bar`. Nesting an `alive_bar` inside another, or inside another tool's live display, is not supported.
- **It needs a real TTY.** In non-interactive contexts — piped output, many CI logs, some containerized runners — terminal size and cursor control are unavailable, and the animation degrades or must be forced. There is explicit handling and a `force_tty` option for PyCharm, Jupyter, and similar; Jupyter is auto-detected but the maintainer has flagged occasional visual glitches there as experimental.
- **Single bar only.** There is no nested/stacked-bar API. Work that fans out into concurrent subtasks each wanting their own bar is out of scope; reach for `enlighten` or `rich` instead.
- **Throughput calibration.** The refresh is capped (around 60 FPS) and the animation speed maps to observed rate. For extreme throughputs or, conversely, very slow long-running jobs (hours, on a k8s pod), use the FPS calibration and custom-fps settings so the animation and ETA stay meaningful.
- **Resuming skews ETA unless you tell it.** When skipping already-done items on a restart, the bar would otherwise infer an enormous rate. Since 3.1 you pass `bar(skipped=True)` (or `bar(N, skipped=True)`) so skipped items do not poison the rate/ETA estimate.
- **Ctrl-C behavior is configurable and has a footgun.** `ctrl_c=False` makes interactive use smoother (no traceback on stop) but inside a loop it continues to the next iteration rather than aborting, which may not be what you want.
- **Type checking.** `py.typed` ships as of the 3.3 series, so `mypy` and other checkers see inline types without stubs.

## When to Use / When Not

**Use when:**
- You have one long-running loop in a script, CLI, or REPL session and want a clear, engaging in-progress signal plus an accurate ETA.
- You want print/logging output to interleave cleanly above the bar with no manual coordination.
- You value the interactive extras: pause/resume back to the prompt, a final receipt, over/underflow counting, and heavy visual customization.

**Avoid when:**
- You need multiple or nested concurrent progress bars — use `rich` or `enlighten`.
- The bar must live inside a larger TUI layout (tables, panels, live regions) — use `rich`.
- Output is non-interactive (CI logs, piped, redirected to a file) where an animation is noise rather than signal.
- You want the most ubiquitous, lowest-friction dependency with pandas/notebook integration — `tqdm` is the safer default.

## Alternatives

- tqdm/tqdm — the de facto standard; minimal overhead, nested bars, pandas/`map` integration. Use when you want ubiquity, nesting, or the smallest surface area.
- Textualize/rich — full-featured terminal toolkit whose `Progress` supports multiple concurrent tasks alongside tables and panels. Use when the bar is part of a larger rich UI.
- Rockhopper-Technologies/enlighten — designed for multiple/nested bars with logging that never clobbers the display. Use when you need several bars plus clean log output.
- WoLpH/python-progressbar — long-standing configurable widget-based bar. Use when you want a mature, no-frills alternative without the stdout takeover.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019 | Initial release: reactive spinner, ETA, print hook[^1]. |
| 2.0 | 2021 | Cell-architecture rewrite: emoji/grapheme support, spinner compiler, `alive_it` adapter[^2]. |
| 2.1 | 2021 | Jupyter auto-detection, disabled state. |
| 2.4 | 2022 | Dual-line mode; `finalize` for the receipt. |
| 3.0 | 2022 | Units, SI/IEC auto-scaling, configurable precision, `sys.stderr` support. |
| 3.1 | 2023 | Resume support via `bar(skipped=True)`; `max_cols` config. |
| 3.2 | 2024 | Multithreaded print/logging hooks; dropped Python 3.7/3.8. |
| 3.3 | 2025 | Receipt accessible from the handle; `graphemeu` dependency; `py.typed`; Python 3.14 in CI[^3]. |

## References

[^1]: alive-progress on PyPI — release history and metadata. https://pypi.org/project/alive-progress/#history
[^2]: alive-progress README, "New in 2.0 series" — cell architecture, spinner compiler, `alive_it`. https://github.com/rsalmei/alive-progress#readme
[^3]: alive-progress README, "What's new in 3.3 series" — receipt handle access, `graphemeu`, `py.typed`, Python 3.14. https://github.com/rsalmei/alive-progress#-whats-new-in-33-series

## Tags

python, cli, terminal, progress-bar, spinner, eta, tui, throughput, console, developer-tools
