# tqdm/tqdm

> A progress-bar wrapper for Python iterables that adds an ETA meter to any loop with ~60ns of per-iteration overhead.

[GitHub repo](https://github.com/tqdm/tqdm) ·
[Official website](https://tqdm.github.io) ·
[License: MPL-2.0 AND MIT](https://github.com/tqdm/tqdm/blob/master/LICENCE)

## Overview

tqdm wraps any iterable and prints a live, self-updating progress bar to the
terminal as the loop consumes it. The name derives from the Arabic *taqaddum*
("progress")[^1]. The entire premise is that adding a progress meter should cost
almost nothing at the call site (`for x in tqdm(iterable):`) and almost nothing
at runtime — the library reports roughly 60ns per iteration versus ~800ns for
the older `python-progressbar`[^2]. It has no dependencies beyond the standard
library, works on any platform and any console that honors carriage return, and
degrades to an ASCII bar where Unicode is unavailable.

The defining tension in tqdm is that a progress bar is fundamentally an I/O and
timing concern grafted onto a hot loop. To stay cheap it does not redraw on
every iteration; it throttles display to `mininterval` seconds (default 0.1) and,
with `dynamic_miniters`, learns how many iterations to skip between redraws so
the timing check itself does not dominate a tight loop. This throttling logic —
plus a background monitor thread, ANSI cursor control for nested bars, and an
EMA-smoothed rate estimator — is most of the library's actual complexity. The
API surface (`tqdm(iterable)`) hides it well until you hit a console that does
not behave the way tqdm assumes.

tqdm is one of the most widely depended-on packages in the Python ecosystem,
pulled in transitively by pandas workflows, ML training loops (Keras, PyTorch,
Hugging Face), Jupyter notebooks, and countless CLI tools. Maintenance is
concentrated heavily on a single author, Casper da Costa-Luis, and the project
runs at a deliberate low-churn cadence — the codebase has been on the 4.x series
since 2016 and treats API stability as a feature.

## Getting Started

```bash
pip install tqdm          # or: conda install -c conda-forge tqdm
```

```python
from tqdm import tqdm
from time import sleep

for item in tqdm(range(10000)):
    sleep(0.001)
# 76%|████████████████████        | 7568/10000 [00:33<00:10, 229.00it/s]
```

Manual control when there is no single iterable to wrap (e.g. streaming reads):

```python
with tqdm(total=file_size, unit="B", unit_scale=True) as pbar:
    for chunk in stream:
        process(chunk)
        pbar.update(len(chunk))
```

As a shell filter — pass stdin through to stdout while metering on stderr:

```sh
seq 9999999 | tqdm --bytes | wc -l
# 75.2MB [00:00, 217MB/s]
```

## Architecture / How It Works

tqdm is a single decorating class. `tqdm(iterable)` returns an object whose
`__iter__` yields straight from the underlying iterable and, on each step,
increments an internal counter `n` and *conditionally* triggers a redraw. The
condition is the core of the design:

- **Display throttling.** A redraw happens only when both `mininterval` seconds
  have elapsed *and* `miniters` iterations have passed since the last one. With
  the default `dynamic_miniters`, tqdm continuously tunes `miniters` upward so
  that redraws land near `mininterval` — this is what keeps the per-iteration
  cost at tens of nanoseconds in fast loops, because most iterations skip the
  `time()` call entirely once the stride is learned.
- **Rate / ETA.** Instantaneous speed is smoothed with an exponential moving
  average (`smoothing=0.3` by default) to keep the it/s figure and remaining-time
  estimate from jittering. `smoothing=0` gives cumulative-average speed; `1`
  gives raw instantaneous.
- **Rendering.** The bar string is assembled from `bar_format` fields
  (`{l_bar}{bar}{r_bar}`) and written to `sys.stderr` with a leading `\r` so the
  next write overwrites the current line. There is no full-screen or curses
  layer — just carriage return and, for nested bars, ANSI cursor-up sequences.
- **Monitor thread.** A single background `TMonitor` thread exists to enforce
  `maxinterval`: if a redraw hasn't happened in too long (a slow iteration
  stalled the bar), it forces one and readjusts `miniters` downward.
- **Concurrency.** A class-level lock serializes writes so multiple bars (across
  threads or processes) don't interleave. `position=` assigns each bar a line
  offset for multi-bar output.

Everything else is submodules layered on this core. `tqdm.auto` picks the right
frontend at import time (ipywidgets in a notebook, terminal otherwise) and is the
recommended import for portable code. `tqdm.notebook` renders an HTML widget;
`tqdm.gui` uses matplotlib; `tqdm.rich` delegates drawing to Rich. `tqdm.pandas()`
monkeypatches pandas to add `progress_apply`. `tqdm.contrib` holds integrations:
`concurrent.futures` helpers (`thread_map`/`process_map`), `tqdm.contrib.telegram`
and `discord` bots, plus `itertools`/`enumerate`/`zip` wrappers. The CLI
(`python -m tqdm`) is a thin stdin→stdout pump around the same class. Defaults can
be overridden from the environment (`TQDM_MININTERVAL`, `TQDM_DISABLE`, etc.) via
the reusable `tqdm.utils.envwrap` decorator.

## Production Notes

**Console assumptions are the #1 source of bug reports.** tqdm requires working
carriage-return support; consoles that log line-buffered — CloudWatch, some
Kubernetes log collectors, CI systems — turn a single updating bar into thousands
of stacked lines. Mitigations that actually work: `export TQDM_POSITION=-1` for
`\r`-broken sinks, and `disable=None` to auto-disable when stderr is not a TTY.
For CI noise generally, `export TQDM_MININTERVAL=5` throttles redraws to once
every few seconds.

**Windows and Unicode.** Windows consoles frequently mis-render the smooth block
glyphs (either as double-width or not at all), which breaks alignment; the
documented fix is `ascii=True`. Nested bars on Windows may additionally need the
`colorama` package installed to keep each bar on its own line.

**Generator length is invisible.** Wrapping a generator with tqdm gives no total
and therefore no percentage or ETA — tqdm can't know a generator's length.
`tqdm(enumerate(x))` and `tqdm(zip(a, b))` are common mistakes: rewrite as
`enumerate(tqdm(x))` / `zip(tqdm(a), b)`, or pass `total=len(x)` explicitly.

**Multiprocessing.** Bars from separate processes need explicit `position=` and a
shared lock (`tqdm.set_lock(...)` / `initializer=`) or they clobber each other's
lines. `tqdm.contrib.concurrent.process_map` handles the common case.

**Overhead is low but not zero.** The 60ns figure assumes the fast path where
redraws are being skipped. Very tight numeric loops (tens of millions of trivial
iterations) can still show tqdm in a profile; wrap at a coarser granularity or
raise `miniters`. The `--update`/`--update_to` CLI modes decode every stdin line
as a number and are slow (~2e5 it/s) by comparison.

**Stability and bus factor.** Upgrades within 4.x are low-risk — the public API
has been stable for years, which is a genuine operational advantage. The flip
side is that development is slow-moving and effectively centered on one
maintainer, so novel feature requests and niche console bugs can sit open for a
long time. Treat tqdm as mature infrastructure, not an actively evolving one.

## When to Use / When Not

**Use when:**
- You want a progress meter on a loop, download, or file read with one line and
  no config.
- You need it to work identically across terminal, Jupyter, and shell pipes
  (`tqdm.auto`).
- You care about per-iteration overhead in a long-running batch or training job.
- You want zero added dependencies.

**Avoid when:**
- You need a full-screen dashboard, tables, spinners, and log panes together —
  Rich or Textual are built for that.
- You're logging to a `\r`-hostile sink (some cloud log aggregators) where any
  in-place bar becomes line spam — log a periodic percentage instead.
- You want rich concurrent multi-bar layouts with interleaved log lines —
  enlighten is purpose-built for that.

## Alternatives

- Textualize/rich — progress bars plus tables, syntax highlighting, and live
  layouts; use when the bar is one part of a richer terminal UI, at the cost of a
  dependency and more overhead.
- Rockhopper-Technologies/enlighten — use when you need progress bars and normal
  logging output to coexist cleanly without clobbering each other.
- rsalmei/alive-progress — use when you want animated, visually elaborate bars for
  interactive scripts rather than minimal overhead.
- WoLpH/python-progressbar — the older progressbar2; use only for legacy code
  already built around its API.
- pallets/click — use its `click.progressbar` when you're already inside a Click
  CLI and don't want a separate dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-06 | Initial public release; repo created on GitHub[^3]. |
| 4.0 | 2016-11 | Start of the long-lived 4.x series; still the current major line. |
| 4.x | 2018–2020 | `tqdm.auto`, `tqdm.contrib`, pandas `progress_apply`, notebook/gui frontends added over the series. |
| 4.67.x | 2024 | Recent maintenance releases on the same 4.x line. |

## References

[^1]: tqdm README — name etymology (*taqaddum*, "progress"). https://github.com/tqdm/tqdm#readme
[^2]: tqdm README — overhead comparison vs python-progressbar (~60ns vs ~800ns per iteration). https://github.com/tqdm/tqdm#readme
[^3]: tqdm repository metadata — created 2015-06-03. https://github.com/tqdm/tqdm

## Tags

python, cli, progress-bar, terminal, jupyter, tui, iterator, data-science, utilities, zero-dependency, console
