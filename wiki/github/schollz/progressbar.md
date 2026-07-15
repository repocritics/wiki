# schollz/progressbar

> A thread-safe, single-line terminal progress bar for Go that doubles as an `io.Writer`.

[GitHub repo](https://github.com/schollz/progressbar) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/schollz/progressbar/v3?tab=doc) ·
[License: MIT](https://github.com/schollz/progressbar/blob/main/LICENSE)

## Overview

`progressbar` is a small Go library that renders a single-line progress indicator to a terminal. It was written by Zack Scholl (schollz) as a dependency for [croc](https://github.com/schollz/croc), his file-transfer tool, after existing bars misbehaved across platforms[^1]. The stated design goal is to work on every OS without special handling, which drives its most consequential constraint: it deliberately does not support multi-line output[^2].

The library is now on its third major version (`/v3`), imported via the versioned module path. It is widely used across the Go CLI ecosystem — several thousand dependents — precisely because it is small, has few transitive dependencies, and does one thing. It is not a TUI framework and does not try to be; there is no layout engine, no concurrency of multiple bars, no event loop. The defining tradeoff is scope: you get a bar that is trivial to drop in and hard to misuse, at the cost of anything beyond one line.

Its most-used trick is that the bar implements `io.Writer`. This lets it sit inside an `io.Copy` or `io.MultiWriter` pipeline and advance itself by counting bytes, which is how most download/upload progress in Go CLIs is wired without any manual `Add` bookkeeping.

## Getting Started

```bash
go get -u github.com/schollz/progressbar/v3
```

```go
package main

import (
	"time"

	"github.com/schollz/progressbar/v3"
)

func main() {
	bar := progressbar.Default(100)
	for i := 0; i < 100; i++ {
		bar.Add(1)
		time.Sleep(40 * time.Millisecond)
	}
}
```

Byte-counting mode, the common case for downloads, uses the `io.Writer` interface directly:

```go
bar := progressbar.DefaultBytes(resp.ContentLength, "downloading")
io.Copy(io.MultiWriter(f, bar), resp.Body)
```

If `ContentLength` is `-1` (unknown total), the bar automatically renders as a spinner instead.

## Architecture / How It Works

The core is a single `ProgressBar` struct guarded by a mutex; every mutating call (`Add`, `Set`, `Describe`, render) takes the lock, which is what "thread-safe" means here — you can advance the same bar from multiple goroutines without corrupting the line. Configuration is done through functional options (`progressbar.NewOptions(max, OptionSetWidth(...), OptionSetTheme(...), ...)`); the `Default` and `DefaultBytes` constructors are just opinionated bundles of those options.

Rendering works by writing the whole line, then emitting a carriage return (`\r`) to move the cursor back to column zero so the next frame overwrites in place. This is the mechanism that keeps the bar to one line and makes it portable — it never uses cursor-addressing escape sequences that vary by terminal. It is also exactly why multi-line is unsupported: `\r` can only rewind the current line, so there is no OS-agnostic way to redraw N lines above the cursor[^2].

The visual is composed from a `Theme` struct — `Saucer` (filled portion), `SaucerHead` (the leading character), `SaucerPadding` (empty portion), and `BarStart`/`BarEnd` delimiters. Color is not computed by the library; you pass ANSI-coded strings in the theme and enable `OptionEnableColorCodes(true)`, and the README notes that on Windows you should route the writer through `github.com/k0kubun/go-ansi` so escape codes are interpreted rather than printed literally. Throttling (`OptionThrottle`) limits how often the line is repainted regardless of how often you call `Add`, which matters when you advance the bar in a hot loop.

## Production Notes

- **Writer choice governs correctness.** By default the bar writes to `os.Stderr`. Mixing bar output with your own `stdout`/`stderr` logging interleaves carriage-returns with log lines and produces garbled scrollback. In practice you either send the bar to stderr and data to stdout, or suppress logging while the bar is live.
- **Not a TTY? It still writes.** The bar does not universally auto-detect a pipe; if you leave it rendering when output is redirected to a file or captured in CI, you can get a file full of `\r`-laden frames. Gate bar creation on `term.IsTerminal(...)` (or use `OptionSetVisibility`) when output may not be interactive.
- **Width and wrapping.** If the rendered line exceeds the terminal width it wraps to a second physical line, and the `\r` trick then only rewinds the last wrapped line, leaving a trail of stale frames. Set an explicit `OptionSetWidth` or use `OptionFullWidth` deliberately rather than letting long descriptions overflow.
- **Throttle in tight loops.** Calling `Add(1)` millions of times without `OptionThrottle` makes rendering, not your work, the bottleneck. A throttle of ~65ms is typical.
- **Unknown-length spinner never "completes" cleanly.** When `max == -1` the bar is a spinner with no percentage; you are responsible for calling `Finish`/`Close` to stop it and print a newline, otherwise the next output lands on the spinner line.
- **v1 → v2 → v3 import paths.** The module path is versioned (`/v3`). Upgrading across majors is a code change (import path plus API adjustments), not just a `go get`, and old tutorials frequently reference the unversioned or `/v2` API. Pin to `/v3` for new code.

## When to Use / When Not

**Use when:**
- You want a one-line percentage or byte-count bar in a CLI with near-zero setup.
- Your progress is naturally an `io.Reader`/`io.Writer` stream (downloads, copies, hashing).
- You need portability across Linux/macOS/Windows without terminal-capability probing.
- You want minimal dependencies and a small surface you can reason about.

**Avoid when:**
- You need multiple concurrent bars, nested progress, or full-screen layout — the library explicitly won't do this.
- You want a rich TUI (tables, panels, live-updating regions) — reach for a framework.
- You need fine control over terminal state on exotic terminals; the `\r`-based approach is intentionally lowest-common-denominator.

## Alternatives

- vbauerster/mpb — use when you need multiple progress bars rendered concurrently with a bar-group/decorators model.
- charmbracelet/bubbletea — use when the bar is one piece of a larger interactive full-screen TUI rather than a standalone line.
- cheggaaa/pb — use when you want a long-established bar with a similar single-line scope and a mature option set.
- pterm/pterm — use when you want progress as part of a broader styled-output/printing toolkit.
- briandowns/spinner — use when you only need an indeterminate spinner, not a measured bar.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2017-10-26 | Created as a dependency for croc[^1]. |
| v2.0 | 2019 | Major revision; substantial improvements credited to @Dynom in the README[^3]. |
| v3.x | ongoing | Current major line; versioned module path `/v3`, spinner mode, functional options, byte-counting `DefaultBytes`. |

Exact per-release dates are not restated here because the project tags frequently within v3; consult the repository's releases/tags for the current patch level[^4]. As of this writing the repo carries roughly 4.7k stars and ~250 forks, sees ongoing commits, and is not archived — it is maintained but stable rather than fast-moving.

## References

[^1]: README — origin as a dependency for croc, motivated by cross-platform issues with existing bars. https://github.com/schollz/progressbar#readme
[^2]: Issue #6 — maintainer position that multi-line output is out of scope to preserve OS-agnostic behavior. https://github.com/schollz/progressbar/issues/6
[^3]: README "Thanks" section crediting v2.0 improvements. https://github.com/schollz/progressbar#thanks
[^4]: Releases and tags. https://github.com/schollz/progressbar/releases

## Tags

go, golang, cli, terminal, progress-bar, spinner, io-writer, thread-safe, library, command-line
