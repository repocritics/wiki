# spf13/pflag

> Drop-in replacement for Go's standard `flag` package that implements POSIX/GNU-style `--long` and `-s` short flags.

[GitHub repo](https://github.com/spf13/pflag) ·
[License: BSD-3-Clause](https://github.com/spf13/pflag/blob/master/LICENSE)

## Overview

pflag is a command-line flag parsing library for Go. Its selling point is
compatibility with the [GNU extensions to POSIX option syntax][gnu]: `--flag`,
`--flag=value`, combined single-character shorthands (`-abc`), and flags
interspersed with positional arguments. Go's standard library `flag` package
deliberately does none of this — it uses single-dash `-flag`, stops parsing at
the first non-flag argument, and has no concept of a short/long pair. pflag
exists to fill that gap while remaining source-compatible enough that importing
it as `flag "github.com/spf13/pflag"` leaves most existing code working[^1].

Its star count (low thousands) badly understates its reach. pflag is the flag
engine underneath spf13/cobra, which in turn powers kubectl, Helm, Hugo, GitHub
CLI, containerd, and a large share of the Go CLI ecosystem. For most projects
pflag is a transitive dependency pulled in by cobra rather than a direct choice,
so its real deployment footprint is far larger than its own popularity suggests.
It originated as a fork of ogier/pflag and has been maintained under spf13 since
around 2013[^2].

The defining tension is stability versus staleness. pflag's API has been frozen
in practice for years — this is a virtue for a library sitting under half the Go
CLI world, but it also means open issues accumulate and the parser's quirks
(single-dash semantics, `go test` flag handling) are permanent rather than
things a future version will smooth over.

## Getting Started

```bash
go get github.com/spf13/pflag@latest
```

```go
package main

import (
	"fmt"

	flag "github.com/spf13/pflag"
)

func main() {
	// long + short in one call: --verbose / -v
	verbose := flag.BoolP("verbose", "v", false, "verbose output")
	name := flag.StringP("name", "n", "world", "who to greet")

	flag.Parse()

	if *verbose {
		fmt.Println("verbosity on")
	}
	fmt.Printf("hello, %s\n", *name)
	fmt.Println("remaining args:", flag.Args())
}
```

```
$ ./greet -v --name=Tom extra-arg
verbosity on
hello, Tom
remaining args: [extra-arg]
```

Every flag constructor has a `P` variant (`StringP`, `IntVarP`, `VarP`) that adds
a one-letter shorthand. Without the `P`, you get a long flag only.

## Architecture / How It Works

The core type is `FlagSet`, a named collection of flags with its own error
handling policy (`ContinueOnError`, `ExitOnError`, `PanicOnError`). The
package-level functions (`flag.String`, `flag.Parse`) operate on a global
`CommandLine` FlagSet, mirroring the standard library. Subcommands are built by
creating independent FlagSets — this is exactly how cobra assigns a flag set per
command.

Each flag is a `*Flag` struct (name, shorthand, usage, default, current value,
and metadata like `NoOptDefVal`, `Deprecated`, `Hidden`, `Changed`). Values are
stored behind the `Value` interface:

```go
type Value interface {
	String() string
	Set(string) error
	Type() string
}
```

The `Type()` method is the one incompatibility with `flag.Value` from the
standard library — pflag needs it to render typed usage strings (`--count int`).
This means a stdlib `flag.Value` implementation does not automatically satisfy
`pflag.Value`; custom flag types written against the standard library need a
`Type()` method added.

Parsing walks `os.Args`, distinguishing three token shapes: `--long` (long
flags), `-x` (one or more shorthands), and `--` (the terminator, after which
everything is positional). Single-dash tokens are treated as a *series* of
shorthands, so `-abc` means `-a -b -c` where all but the last must be boolean or
carry a `NoOptDefVal`. This is why `-b true` is invalid — the parser reads
`true` as the next argument, not the boolean's value; you must write `-b=true`.
Flags and positional arguments may be freely interleaved before `--`.

Two extension mechanisms shape real usage: a `NormalizedName` function
(`SetNormalizeFunc`) can canonicalize names so `--my-flag`, `--my_flag`, and
`--my.flag` compare equal or so aliases resolve to one canonical flag; and
`AddGoFlagSet` splices a standard-library `flag.FlagSet` into a pflag set so that
flags registered by third-party libraries (classically `glog`) still parse.

## Production Notes

**Migration is not literally drop-in.** The claim holds for code that uses the
constructor functions, but two behaviors differ from the standard library and
will surprise people: interspersed flags/args (stdlib stops at the first
positional; pflag does not), and single-dash semantics (`-verbose` in pflag is
parsed as the shorthands `-v -e -r -b -o -s -e`, not the long flag `verbose`).
Code that relied on stdlib's "stop at first non-flag" behavior can break subtly.

**`go test` flags are not fully parsed.** pflag does not understand go test's
`-test.*` shorthands, so calling `pflag.Parse()` inside `TestMain` can silently
drop flags like `-v`[^3]. The workaround is `ParseSkippedFlags`, which hands the
skipped flags to the standard `flag` package. This is a recurring source of
confusion (issues #63, #238).

**Boolean shorthand ergonomics.** `-b=true` works; `-b true` does not (the space
form only applies to non-boolean flags). Boolean long flags accept
`1/0/t/f/true/false` and case variants; short booleans combine (`-abc`).

**NoOptDefVal is subtle.** Setting `Lookup("flag").NoOptDefVal` changes a flag so
that supplying it bare (`--flag`) yields the no-opt value while `--flag=x` still
takes `x`. It is what makes count-style and optional-value flags work, but it
must be set after the flag is defined and is easy to forget when the flag is
created deep inside a helper.

**Maintenance cadence.** Releases are infrequent and the API is effectively
frozen. That stability is deliberate and appropriate given how much depends on
it, but do not expect open issues to be resolved quickly, and do not expect new
POSIX corner cases to be added. Pin the version; upgrades are rarely urgent.

**Sorting.** `SortFlags` is a field on `FlagSet`, not a package global. Disabling
alphabetical ordering for the default set requires `pflag.CommandLine.SortFlags
= false`, not a top-level toggle.

## When to Use / When Not

**Use when:**
- You want GNU/POSIX flag syntax (`--flag`, `-f`, `--flag=value`) in Go.
- You need long/short flag pairs, interspersed args, or subcommand-scoped flag
  sets.
- You are already using cobra or viper (you are using pflag whether you chose it
  or not).
- You want flag deprecation, hidden flags, or custom name normalization.

**Avoid when:**
- Standard-library `flag` syntax is fine and you want zero dependencies.
- You need a full command framework with help generation and nested subcommands
  — reach for cobra (which wraps pflag) rather than pflag alone.
- You want struct-tag-driven or declarative flag definition — pflag is
  imperative only.

## Alternatives

- spf13/cobra — full CLI framework built on top of pflag; use when you need
  subcommands, generated help, and shell completion rather than just parsing.
- urfave/cli — self-contained CLI framework with its own flag layer; use when
  you want commands + flags in one package without the cobra/pflag split.
- alecthomas/kingpin — fluent builder-style flag and command parser; use when
  you prefer method-chaining definitions.
- jessevdk/go-flags — struct-tag-based parsing; use when you want flags declared
  as annotated struct fields instead of function calls.
- standard library `flag` — use when you have no dependency budget and single-
  dash syntax is acceptable.

## History

| Point | Date | Notes |
|-------|------|-------|
| Repo created | 2013-08-30 | Maintained under spf13, forked from ogier/pflag[^2]. |
| v1.0.x series | 2018 onward | Tagged 1.0 releases; API stabilized and effectively frozen. |
| Last push | 2026-07-03 | Still receiving occasional maintenance commits, not archived. |

## References

[^1]: pflag README — Description and Usage. https://github.com/spf13/pflag#readme
[^2]: pflag repository, created 2013-08-30 under spf13. https://github.com/spf13/pflag
[^3]: pflag README — "Using pflag with go test"; see issues #63 and #238. https://github.com/spf13/pflag/issues/63

## Tags

go, cli, command-line, flag-parsing, posix, gnu-getopt, argument-parsing, cobra, library, spf13
