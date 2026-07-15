# vbauerster/mpb

> Multi progress bar library for Go CLI applications, built around a single render goroutine (actor model) instead of shared locks.

[GitHub repo](https://github.com/vbauerster/mpb) ·
[License: Unlicense](https://github.com/vbauerster/mpb/blob/master/UNLICENSE)

## Overview

`mpb` renders one or many progress bars in a terminal from a Go program. It has
been maintained by Vyacheslav Bauer since 2016[^1] and is one of the two or three
libraries reached for when a Go CLI needs concurrent progress display — downloads,
parallel jobs, multi-file operations — rather than a single bar. As of 2026 it sits
around 2.5k stars and is still actively pushed to, with a small, stable maintainer
surface.

The defining design choice is that a `Progress` container owns a dedicated
rendering goroutine and every mutation (increment, add bar, remove bar, set total)
is delivered to it over channels. Bars do not hold locks that callers contend on;
instead the container serializes state and repaints on a fixed cadence. This is why
the repo carries an `actor` topic. The tradeoff is that the container has a
lifecycle you must respect — you create it, spawn work, and then call `Wait()` to
flush the final frame and stop the goroutine. Forget the `Wait()` and you leak a
goroutine and never see completed bars flushed.

The second tension is API stability. `mpb` uses Go's module-path major versioning
aggressively: v4 through v8 each live at a distinct import path
(`github.com/vbauerster/mpb/v8`) and each major bumped for breaking API changes to
decorators, bar fillers, or the container constructor. Pinning a major and reading
that major's examples matters — answers for `/v6` frequently do not compile under
`/v8`.

## Getting Started

```bash
go get github.com/vbauerster/mpb/v8
```

```go
package main

import (
	"math/rand"
	"time"

	"github.com/vbauerster/mpb/v8"
	"github.com/vbauerster/mpb/v8/decor"
)

func main() {
	p := mpb.New(mpb.WithWidth(64))

	total := 100
	bar := p.New(int64(total),
		mpb.BarStyle().Lbound("[").Filler("=").Tip(">").Padding(" ").Rbound("]"),
		mpb.PrependDecorators(decor.Name("Downloading:")),
		mpb.AppendDecorators(
			decor.OnComplete(decor.Percentage(), "done"),
		),
	)

	max := 100 * time.Millisecond
	for i := 0; i < total; i++ {
		time.Sleep(time.Duration(rand.Intn(10)+1) * max / 10)
		bar.Increment()
	}
	p.Wait() // flush final frame and stop the render goroutine
}
```

`bar.Increment()` and the other bar mutators are safe to call from multiple
goroutines — that is the whole point of the container. For byte-oriented work,
`bar.ProxyReader(r)` wraps an `io.Reader` so reads advance the bar automatically.

## Architecture / How It Works

- **Container as actor.** `mpb.New(...)` starts a goroutine running a render loop.
  Public methods (`AddBar`, `bar.Increment`, `bar.SetTotal`, `bar.Abort`) send
  messages to that loop over channels rather than mutating shared state directly.
  This removes caller-visible lock contention and makes concurrent updates from
  many worker goroutines safe by construction.
- **Refresh cadence.** The loop repaints on a timer (default refresh rate on the
  order of ~150ms, tunable via `mpb.WithRefreshRate`). Increments between frames
  are coalesced, so calling `Increment()` in a tight loop does not cause one redraw
  per call.
- **Terminal writer.** Repainting uses a `cwriter` that emits VT100/ANSI cursor
  movement (move up N lines, clear, rewrite) so an N-bar block is overwritten in
  place each frame. Cell width is computed with `mattn/go-runewidth` so CJK and
  wide glyphs align, and `acarl005/stripansi` is used to measure strings that
  carry color escapes.
- **Decorators.** Each bar has a prepend and an append decorator list. Decorators
  are small functions of bar statistics (`decor.Percentage`, `decor.CountersKibiByte`,
  `decor.EwmaETA`, `decor.Name`, ...). ETA decorators back onto `VividCortex/ewma`
  for a moving average; width-sync (`decor.WCSyncWidth`, `WCSyncSpace`) aligns a
  column across all bars so multi-bar output lines up.
- **Fillers.** `BarStyle()` builds the bar body (left bound, filler, tip, padding,
  right bound); `SpinnerStyle()` builds a spinner. Fillers are an interface, so
  custom renderers are possible.

Dependencies are deliberately few: `golang.org/x/sys`, `go-runewidth`, `ewma`,
`stripansi`. There is no heavyweight TUI runtime underneath.

## Production Notes

- **Non-TTY output is the main footgun.** When stdout is not a terminal (piped to a
  file, a CI log, or through another process), cursor-movement repainting makes no
  sense. `mpb` will not produce sensible in-place output there; use
  `mpb.WithAutoRefresh` for environments that still need periodic frames, or gate
  bar creation on an `isatty` check and fall back to plain logging. Silent garbage
  in CI logs is almost always this.
- **Interleaving with your logger corrupts the display.** Any code that writes to
  the same stream (stderr/stdout) outside the container will tear the rendered
  block. Route log lines through `container.Write()` / `bar.Write` aware paths, or
  send logs to a different stream than the bars.
- **EWMA decorators have a contract.** `decor.EwmaETA` requires you to advance the
  bar with `bar.EwmaIncrement(duration)` (or `DecoratorAverageAdjust`) so it has
  per-iteration timings. Feeding an EWMA decorator with a plain `Increment()` gives
  meaningless or absent ETAs.
- **You must drive completion.** A bar completes when its current count reaches its
  total, or when you call `bar.SetTotal(-1, true)` / `bar.Abort()`. Bars with an
  unknown total (spinner or `-1` total) never auto-complete — you are responsible
  for aborting them, or `Wait()` blocks forever.
- **`Wait()` is mandatory.** It is the flush-and-shutdown call. Deferring it right
  after `mpb.New` is the safe idiom; combine with `mpb.WithWaitGroup(&wg)` so the
  container also waits on your workers.
- **Major-version churn.** Upgrading across majors (e.g. v6→v7→v8) is a source-code
  migration, not a `go get -u`. Decorator constructors, the `WC` width-config
  struct, and bar-filler builders have all changed shape between majors; budget
  time to update call sites and re-read the current examples.

## When to Use / When Not

**Use when:**
- You need several concurrent progress bars from parallel goroutines and want
  correctness without hand-rolling locks.
- You want per-bar ETA / bytes / percentage decorators with column alignment.
- You want a focused dependency, not a full TUI framework, in a CLI.

**Avoid when:**
- You only need one simple bar — `schollz/progressbar` is a smaller, single-file dependency.
- Your output is primarily non-interactive (CI, log files): the in-place rendering
  is wasted and needs guarding.
- You are already building a full-screen TUI — use your framework's own progress
  component instead of layering `mpb` inside it.

## Alternatives

- schollz/progressbar — single bar, minimal API and dependency; use when you do not need concurrency or multi-bar layout.
- cheggaaa/pb — long-standing progress bar (v3), supports pools of bars; use when you prefer its older, widely-used API.
- charmbracelet/bubbles — progress component for the Bubble Tea TUI framework; use when the app is already a full-screen TUI.
- pterm/pterm — broad terminal-output toolkit including progress printers; use when you also want tables, spinners, and styled text from one library.
- briandowns/spinner — spinners only; use when you need an indeterminate indicator rather than measured progress.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2016-12 | Initial release; multi-bar container concept[^1]. |
| v4 | ~2019 | Import path moves to `/v4`; decorator API reworked. |
| v5 | ~2020 | `/v5`; container and decorator refactors. |
| v6 | ~2021 | `/v6`; bar-filler and width-config changes. |
| v7 | ~2022 | `/v7`; further decorator/API adjustments. |
| v8 | current | `/v8`; `BarStyle()`/`SpinnerStyle()` builders, current maintained line[^2]. |

Exact per-major release dates should be read from the GitHub releases/tags page;
the entries above give the ordering and the module-path scheme, which is the part
that actually affects a consumer.

## References

[^1]: vbauerster/mpb repository, created 2016-12-14. https://github.com/vbauerster/mpb
[^2]: mpb v8 package documentation. https://pkg.go.dev/github.com/vbauerster/mpb/v8

## Tags

go, golang, cli, progress-bar, terminal, tui, spinner, concurrency, actor-model, ansi
