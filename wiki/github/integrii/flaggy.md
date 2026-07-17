# integrii/flaggy

> A dependency-free Go command-line parser that puts subcommands, positional values, and flags at any argument position.

[GitHub repo](https://github.com/integrii/flaggy) ·
[Package reference](https://pkg.go.dev/github.com/integrii/flaggy) ·
[License: Unlicense](https://github.com/integrii/flaggy/blob/master/LICENSE)

## Overview

Flaggy is a single-package Go library for parsing command-line input. It occupies
the middle of the Go CLI spectrum: more ergonomic than the standard library's
`flag` package, but deliberately smaller than framework-style tools like Cobra.
Its stated design goals are zero external dependencies and no imposed project or
package layout — you call functions on a global parser (or your own) and it
writes parsed values directly into variables you already declared[^1].

The library's defining choice is that flags may appear at any position, including
interleaved with positional arguments and subcommands. This is friendlier to end
users (who rarely remember flag ordering rules) but means Flaggy parses the whole
argument vector itself rather than deferring to conventions; the tradeoff is a
parser that is more permissive and slightly less predictable about ambiguous
input than the strict left-to-right `flag` package. It supports both single- and
double-dash forms (`-f`, `--f`, `-flag`, `--flag`), `=` or space assignment, and
`--` to terminate parsing and collect trailing arguments[^1].

At roughly 951 stars and 33 forks it is a modestly popular, stable utility rather
than a fast-moving project — the sort of dependency teams pick precisely because
it is small and does not pull in a tree of transitive packages. It is public
domain via the Unlicense, so there are effectively no attribution obligations.

## Getting Started

```bash
go get -u github.com/integrii/flaggy
```

```go
package main

import "github.com/integrii/flaggy"

func main() {
	// Declare the variable a flag will populate, with its default.
	stringFlag := "defaultValue"

	// Register a flag: short name, long name, description.
	flaggy.String(&stringFlag, "f", "flag", "A test string flag")

	// Parse os.Args into the registered variables.
	flaggy.Parse()

	println(stringFlag) // set from -f / --flag, else the default
}
```

Subcommands attach at an expected positional slot, and can nest:

```go
sub := flaggy.NewSubcommand("build")
sub.String(&target, "t", "target", "Build target")
flaggy.AttachSubcommand(sub, 1) // expected at argument position 1
flaggy.Parse()
if sub.Used { /* "build" was invoked */ }
```

## Architecture / How It Works

Flaggy centers on two types: `Parser` and `Subcommand`. A package-level
`DefaultParser` singleton backs the top-level helper functions (`flaggy.String`,
`flaggy.Bool`, `flaggy.Parse`, etc.), mirroring the standard library's global-
`flag` ergonomics. You can instead construct your own `Parser` for isolation
(useful in tests or libraries), since the global state is otherwise process-wide.

Flag registration is by typed method rather than reflection: there is a distinct
function per supported type (`String`, `Int64`, `Duration`, `IP`, and so on),
each taking a pointer to a pre-declared variable. Flaggy stores that pointer and
assigns through it during parsing. This is why defaults are simply the variable's
existing value — Flaggy never resets what it does not see. The library advertises
35 flag types, covering the basic Go scalars and their slice forms plus a set of
standard-library structures: `time.Duration`/`time.Time`, `net.IP` and friends,
the newer `netip.Addr`/`Prefix`/`AddrPort`, `url.URL`, `regexp.Regexp`,
`big.Int`/`big.Rat`, `os.FileMode`, and a base64 `[]byte` helper[^1].

Parsing is a manual scan of the argument vector rather than a grammar-driven
pass. Because any flag can appear at any position, Flaggy walks the arguments,
resolves subcommands at their attached positions, associates flags with the
active (sub)command scope — global flags remain visible inside subcommands — and
collects positional variables. Everything after a bare `--` is captured verbatim
into `TrailingArguments`. Subcommands can be nested by attaching a subcommand to
another subcommand, forming a positional tree.

Two conveniences are built on top: typo suggestions for mistyped subcommands, and
generated shell completion for bash, zsh, fish, PowerShell, and Nushell exposed
through a `completion` subcommand[^1]. Help output is templated and overridable
per command, with optional prepend/append text.

## Production Notes

- **Global mutable state.** The `DefaultParser` and its registered flags are
  package-level. Registering flags from multiple call sites, or parsing more than
  once in a long-lived process, mutates shared state. For test suites or embedding
  Flaggy inside a library consumed by others, construct an explicit `Parser`
  instead of using the global helpers.
- **Defaults are whatever the variable already holds.** Because Flaggy assigns
  through your pointer only when a flag is present, an uninitialized variable is
  your default. This makes environment-variable defaults trivial (assign
  `os.Getenv(...)` before registering) but also means a stale variable value is
  silently the fallback.
- **Permissive by design.** Flags at any position and lenient dash handling are a
  UX win but reduce strictness. If you need a parser to reject unusual orderings
  or unknown-flag shapes, tune behavior via parser fields such as
  `ShowHelpOnUnexpected` rather than assuming rejection by default.
- **Version injection is a build-time convention.** The idiomatic pattern is a
  `var version = "unknown"` overwritten with `go build -ldflags='-X main.version=...'`
  and passed to `flaggy.SetVersion`. There is no automatic version discovery.
- **No stability of the global on re-parse.** Calling `Parse()` a second time in
  the same process is not the intended usage; per-invocation parsing in a CLI
  binary is the supported model.

## When to Use / When Not

**Use when:**
- You want a small, dependency-free parser with subcommands and positional args.
- You value user-forgiving argument ordering over strict grammar.
- You want built-in shell completion and readable help without extra libraries.
- You prefer assigning into your own variables over a config struct or context.

**Avoid when:**
- You need a large command framework with generated scaffolding, command trees,
  and an ecosystem of plugins — Cobra is the ecosystem standard there.
- You want struct-tag / reflection-based declarative parsing (`go-flags`).
- You need POSIX/GNU getopt-exact semantics and strict validation.
- You are wiring into Viper-style layered configuration; Flaggy is parse-only.

## Alternatives

- spf13/cobra — use when you want the de facto command framework with generators, command trees, and a plugin ecosystem, and can accept its heavier structure and dependencies.
- spf13/pflag — use when you only need POSIX/GNU-style flags as a near drop-in for the standard `flag` package, without subcommands.
- urfave/cli — use when you want a batteries-included app-and-command model that is lighter than Cobra but still framework-shaped.
- jessevdk/go-flags — use when you prefer declaring flags via struct tags and reflection rather than function calls.
- alecthomas/kingpin — use when you want a fluent builder API with strong validation and typed terminators.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2018-03-05 | Repository created; stdlib-only parser with subcommands and any-position flags[^2]. |
| 1.x series | 2018–2021 | Stabilized global/subcommand flags, positional values, nested subcommands, typo suggestions. |
| Recent | 2024–2025 | Expanded type coverage (`netip` types, more stdlib structures) and Nushell shell completion[^1]. |
| Last activity | 2025-10-04 | Most recent push to `master`; low open-issue count, maintenance-mode cadence[^2]. |

Per-release version numbers and dates are not individually verified here; the
anchors above are from repository metadata and the current README feature set.

## References

[^1]: flaggy README — features, supported flag types, and shell completion. https://github.com/integrii/flaggy/blob/master/README.md
[^2]: GitHub repository metadata (creation and last-push dates, license, stars/forks) via the GitHub REST API — retrieved 2026-07. https://github.com/integrii/flaggy

## Tags

go, cli, command-line, argument-parsing, flags, subcommands, positional-arguments, zero-dependency, unlicense, library
