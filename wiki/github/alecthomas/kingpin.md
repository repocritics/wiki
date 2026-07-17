# alecthomas/kingpin

> A fluent, type-safe command-line and flag parser for Go — feature-stable, and explicitly in contributions-only maintenance since its author moved on to Kong.

[GitHub repo](https://github.com/alecthomas/kingpin) ·
[GoDoc](https://pkg.go.dev/github.com/alecthomas/kingpin/v2) ·
[License: MIT](https://github.com/alecthomas/kingpin/blob/master/COPYING)

## Overview

Kingpin is a Go library for parsing command-line flags, positional arguments, and arbitrarily nested subcommands. Its distinguishing trait is a fluent, type-safe builder API: you declare a flag and terminate the chain with a type method (`.Int()`, `.Duration()`, `.ExistingFile()`), which returns a typed pointer that Kingpin populates during `Parse()`. This eliminates the manual `flag.IntVar(&x, ...)` boilerplate of the standard library and gives you validated, converted values without post-parse type assertions. It also generates help output, man pages, and shell completion scripts out of the box.

The critical thing to know before adopting Kingpin in 2026 is its maintenance posture. The README leads with a "CONTRIBUTIONS ONLY" banner: the author, Alec Thomas, states he no longer uses Kingpin personally — he now uses [Kong](https://github.com/alecthomas/kong), his own successor library — and will only merge fixes submitted as PRs[^1]. Kingpin is described by its maintainer as "largely feature stable" with a backlog of known bugs that will not be fixed by the author. It is not abandoned (CI runs, PRs are occasionally merged), but no roadmap exists and design questions will go unanswered.

Despite that, Kingpin remains widely embedded. Prometheus, Alertmanager, and a long tail of Go operational tooling were built on it, which is why it still accrues stars and downloads years after its last feature work. For an existing codebase it is stable and predictable; for a greenfield project it is a library whose own author would steer you elsewhere.

## Getting Started

```bash
go get github.com/alecthomas/kingpin/v2
```

```go
package main

import (
	"fmt"
	"os"

	"github.com/alecthomas/kingpin/v2"
)

var (
	debug   = kingpin.Flag("debug", "Enable debug mode.").Bool()
	timeout = kingpin.Flag("timeout", "Timeout waiting for ping.").
		Default("5s").Envar("PING_TIMEOUT").Short('t').Duration()
	ip    = kingpin.Arg("ip", "IP address to ping.").Required().IP()
	count = kingpin.Arg("count", "Packets to send.").Int()
)

func main() {
	kingpin.Version("0.0.1")
	kingpin.Parse()
	fmt.Printf("ping %s timeout=%s count=%d debug=%v\n", *ip, *timeout, *count, *debug)
	_ = os.Args
}
```

Each `Flag`/`Arg` call returns a settings builder; the terminal type method returns a `*T` you dereference after `Parse()`. For real applications use an explicit `app := kingpin.New(name, help)` instance and dispatch on `app.Command(...).FullCommand()` rather than the package-level default `CommandLine`.

## Architecture / How It Works

Kingpin's public surface is a fluent builder over an internal data model. `Flag()`, `Arg()`, and `Command()` register clauses; each clause holds a `Value` — an interface compatible with the standard library's `flag.Value` (`Set(string) error` and `String() string`). The type methods (`.Int()`, `.Bool()`, `.Duration()`, and dozens more, many code-generated) allocate the appropriate `Value` implementation, wire its target pointer, and return it. Because everything reduces to `flag.Value`, any existing standard-library flag parser works as a custom Kingpin parser unchanged.

Parsing runs in two stages, split apart in the v2 rewrite. `ParseContext()` tokenizes argv into an intermediate context (which command was selected, which flags and args matched) without executing anything; `Parse()` then walks that context, populates the typed pointers, runs validators, and dispatches `Action()` callbacks after all values are set. This separation is what enables features like flags appearing at any point after their command (rather than immediately following it) and shell completion, which needs to reason about the parse tree without side effects.

Repeatability is modeled through an `IsCumulative() bool` method on `Value`. Slice- and map-valued flags, plus `Counter`, return true, which is how both repeated flags (`-v -v -v`) and greedy trailing positional arguments consume multiple tokens through the same mechanism. Help rendering is fully template-driven: usage text is a Go `text/template` you can override (`DefaultUsageTemplate`, `CompactUsageTemplate`, and man-page/completion variants ship built in), and placeholder tokens in help are derived heuristically from `PlaceHolder()`, then `Default()`, then the capitalized flag name.

The v1→v2 boundary is a real API break. `Dispatch()` became `Action()`, `ParseWithFileExpansion()` was removed in favor of built-in `@file` argument expansion, and `ParseContext()`/`Terminate()`/`FatalUsage()` were added[^2]. v1 lives at `gopkg.in/alecthomas/kingpin.v1` and is deprecated; v2 is the only version receiving even contribution-driven fixes.

## Production Notes

- **Package-global default `CommandLine`.** The top-level `kingpin.Flag()`/`kingpin.Parse()` helpers mutate a shared global application. Convenient for a `main`, but it makes flags defined in `init()` or package `var` blocks order-dependent and effectively untestable in isolation. Use an explicit `kingpin.New()` instance in anything larger than a single-file tool.
- **`Parse()` calls `os.Exit` on error.** By default the termination function is `os.Exit`, so a bad argv terminates the process before your `main` continues — inconvenient for tests and for embedding a parser inside a larger program. Override with `app.Terminate(nil)` (or a custom func) and handle the returned error from `app.Parse(args)` yourself.
- **Pointer-return ergonomics vs. struct tags.** Kingpin predates the struct-tag reflection style that Kong and modern Cobra-adjacent libraries use. Defining fifty flags means fifty package-level `var` pointers, which sprawls. There is no first-class "bind this struct" API.
- **Known bugs will stay.** Because the author does not triage issues, edge cases — completion quirks, help-formatting corner cases, some interspersed-argument ambiguities — persist in the tracker. Search closed and open issues before assuming a behavior is intended.
- **Value stability is the upside.** The flip side of frozen maintenance is that Kingpin does not churn. Code written against v2 years ago still compiles and behaves identically, which is exactly why long-lived infra (Prometheus family) never migrated off it.
- **Completion and man pages are genuinely useful.** `--completion-script-bash|zsh|fish` and `--help-man` generate distributable artifacts with no extra dependencies, a feature several alternatives require plugins or extra codegen for.

## When to Use / When Not

**Use when:**
- You maintain an existing Go tool already built on Kingpin — it is stable and there is no forced-migration pressure.
- You want fluent, type-safe flags with built-in man-page and shell-completion generation and no external dependencies.
- You prefer imperative flag declaration over struct-tag reflection.

**Avoid when:**
- You are starting a new project — the author himself recommends Kong, and Cobra dominates the ecosystem for community support.
- You need an active maintainer, a roadmap, or timely bug fixes.
- You want struct-tag-driven binding, or you are building a large CLI (many subcommands) where Cobra's generator and plugin ecosystem pays off.

## Alternatives

- alecthomas/kong — the author's own successor to Kingpin; struct-tag driven, actively maintained. Use when you like Kingpin's philosophy but want a supported library.
- spf13/cobra — the de facto standard for large Go CLIs (kubectl, gh, hugo); use when you need subcommand-heavy tooling, generators, and the widest community.
- urfave/cli — lighter, map/struct-based CLI framework; use when you want something simpler than Cobra without fluent chaining.
- spf13/pflag — POSIX/GNU-style drop-in for the standard `flag` package; use when you only need flags (no subcommands) with getopt-compatible parsing.
- integrii/flaggy — dependency-free, subcommand-capable parser; use when minimal footprint matters more than ecosystem size.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2014-06-19 | First stable release; cumulative positional args, gopkg.in versioning[^3]. |
| v1.2.0 | 2014-07-08 | `Hidden()` flags, `Enum()` value, byte-size unit parser. |
| v1.3.4 | 2015-01-23 | `--` separator, `@file` flag loading, nested subcommands. |
| v2.0.0 | 2015-05-22 | Two-stage parser rewrite; interspersed flags, `Dispatch()`→`Action()`, public data model[^2]. |
| v2.1.0 | 2015-09-19 | `command.Default()`, multi-callback actions, `--help-man`, exposed help/version flags. |
| — | ~2018 | Author adopts Kong; Kingpin enters "contributions only" maintenance[^1]. |

## References

[^1]: Kingpin README, "CONTRIBUTIONS ONLY" notice — the author states he now uses Kong and will only merge submitted PRs. https://github.com/alecthomas/kingpin
[^2]: Kingpin README, "API changes between v1 and v2" and v2.0.0 change history. https://github.com/alecthomas/kingpin#api-changes-between-v1-and-v2
[^3]: Kingpin README, "Change History". https://github.com/alecthomas/kingpin#change-history

## Tags

go, golang, cli, command-line, flag-parser, argument-parsing, subcommands, shell-completion, contributions-only, library
