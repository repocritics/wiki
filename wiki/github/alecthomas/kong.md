# alecthomas/kong

> A struct-tag-driven command-line parser for Go: you declare the CLI as a Go type and Kong maps `os.Args` onto it via reflection.

[GitHub repo](https://github.com/alecthomas/kong) ·
[GoDoc](https://pkg.go.dev/github.com/alecthomas/kong) ·
[License: MIT](https://github.com/alecthomas/kong/blob/master/COPYING)

## Overview

Kong is a Go library for building command-line interfaces where the entire
command tree — commands, sub-commands, flags, positional arguments, and their
help text — is expressed as a Go struct with field tags. It is the spiritual
successor to `alecthomas/kingpin`, the same author's earlier builder-style
parser; Kong replaces kingpin's fluent method chains with declarative struct
tags[^1]. The project has been in production use since 2018 and cut its `1.0.0`
release in 2024, after which the API is considered stable[^2].

The defining tradeoff is declarative concision versus compile-time safety.
A three-line struct can produce a full nested CLI with generated `--help`,
whereas the equivalent in `spf13/cobra` is dozens of lines of imperative
registration. The cost is that the grammar lives in stringly-typed struct tags
(`cmd:""`, `arg:""`, `enum:"a,b,c"`, `required:""`): the Go compiler cannot
check them, so a mistyped tag or an unregistered mapper type surfaces as a
panic or parse error at program startup rather than at build time. Kong suits
teams who value terse CLI definitions and accept runtime-validated grammar;
it is not the choice where the surrounding ecosystem has standardized on Cobra.

## Getting Started

```bash
go get github.com/alecthomas/kong
```

```go
package main

import "github.com/alecthomas/kong"

type RmCmd struct {
	Force bool     `help:"Force removal."`
	Paths []string `arg:"" name:"path" help:"Paths to remove." type:"path"`
}

func (r *RmCmd) Run() error { /* ... */ return nil }

var cli struct {
	Debug bool  `help:"Enable debug mode."`
	Rm    RmCmd `cmd:"" help:"Remove files."`
}

func main() {
	ctx := kong.Parse(&cli)
	ctx.FatalIfErrorf(ctx.Run()) // dispatches to the selected command's Run()
}
```

## Architecture / How It Works

At `kong.Parse()` Kong reflects over the target struct once, building an
internal grammar tree from the field tags. Fields tagged `cmd:""` become
(arbitrarily nestable) commands; `arg:""` fields become positional arguments;
everything else mapped becomes a flag. It then tokenizes the command line and
walks the tree, resolving defaults, applying values, and running validation.

Two dispatch styles coexist. `ctx.Command()` returns a stable string like
`"rm <path>"` you can `switch` on — convenient but fragile, since renaming a
field changes the string. The robust style attaches a `Run(...) error` method
to each leaf command struct; `ctx.Run(bindings...)` then walks from the
selected node back to the root, invoking every `Run` it encounters in reverse
order, which lets parent commands wrap children.

Dependency injection is central. Values passed to `kong.Bind(...)` or through
`ctx.Run(...)` are matched by type into each `Run` method's parameters, so a
command declares the dependencies it needs in its signature rather than reading
globals. Extensibility hangs off a handful of seams: **mappers** (`type:"..."`)
decode strings into arbitrary Go types — builtins include `path`,
`existingfile`, `counter` (for `-vvv`), and `time.Duration`; **resolvers** and
**configuration loaders** feed default values from environment variables or
config files; and **hooks** (`BeforeReset`, `BeforeResolve`, `BeforeApply`,
`AfterApply`, `AfterRun`) let fields intercept the parse lifecycle. The built-in
`--help` flag is itself just a `BeforeReset` hook.

## Production Notes

- **Tags are validated at runtime, not compile time.** A misspelled tag key is
  typically ignored silently; an unknown `type:"..."` mapper or a malformed
  grammar panics on the first `Parse`. Cover CLI construction with a test that
  actually calls `Parse` on representative args, or the failure ships to users.
- **`Parse` exits the process on error.** `kong.Parse` and `FatalIfErrorf` call
  the configured exit function (`os.Exit` by default), which makes them awkward
  to unit-test. For testable parsing, build a parser with `kong.New(&cli)` and
  call `parser.Parse(args)`, or override the exit function with the `kong.Exit`
  option, and assert on the returned error instead.
- **Positional-argument rules bite.** Positionals are required by default
  (add `optional:""`), a trailing slice positional greedily absorbs all
  remaining args, and required positionals must precede optional ones. Getting
  the ordering wrong is a startup panic, not a compile error.
- **Shell completion is not built in.** Unlike Cobra, Kong ships no completion
  generator in the core; the separate `alecthomas/kongplete` package provides
  it. Budget for the extra dependency if completion matters.
- **Struct embedding and `embed:""`/`prefix:""`** flatten nested config into
  namespaced flags (`--logging.level`). This is powerful for sharing flag
  groups but makes the flag namespace non-obvious from any single struct — read
  the whole tree to know what flags exist.
- **Upgrades are low-drama.** The `1.x` line has held API compatibility; the
  one notable break was at `1.0.0` (PR #436), affecting few users[^2]. Minor
  releases have been frequent and additive.

## When to Use / When Not

**Use when:**
- You want the shortest path from a Go struct to a nested CLI with generated help.
- You value type-directed dependency injection into command handlers.
- Your CLI's shape is naturally hierarchical (git/docker-style sub-commands).
- You are coming from kingpin and want the maintained successor.

**Avoid when:**
- Your project or organization has standardized on Cobra (kubectl, Hugo, GitHub
  CLI, and much of the Go ecosystem have) and you want that ecosystem's tooling.
- You need compile-time guarantees about your flag grammar.
- You want batteries-included shell completion and docs generation in core.
- You prefer explicit imperative wiring over reflection and struct tags.

## Alternatives

- spf13/cobra — the de facto Go CLI standard; imperative, verbose, huge ecosystem (completion, docs gen). Use when you want the mainstream toolchain and community.
- urfave/cli — map/builder-style API, lighter than Cobra. Use when you want something simpler than Cobra but not reflection-based.
- alecthomas/kingpin — Kong's fluent predecessor, now in maintenance. Use only for existing kingpin codebases; new work should use Kong.
- jessevdk/go-flags — the other struct-tag parser, closer to getopt semantics. Use when you want tag-driven parsing but prefer its flag conventions.
- spf13/pflag — POSIX/GNU flag primitives (a `flag` replacement), not a command framework. Use when you only need flag parsing and will structure commands yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-04-10 | Repository created; struct-tag parser as kingpin's successor[^1]. |
| 1.0.0 | 2024-08-21 | First stable release; one breaking change (PR #436)[^2]. |
| 1.15.0 | 2026-04-01 | Latest tagged release in the actively-maintained 1.x line. |

## References

[^1]: Kong README — "Introduction"; project positioning relative to kingpin. https://github.com/alecthomas/kong#introduction
[^2]: Kong README — "Version 1.0.0 Release"; stability statement and breaking change #436. https://github.com/alecthomas/kong#version-100-release

## Tags

go, golang, cli, command-line-parser, argument-parsing, struct-tags, reflection, flags, developer-tools, library
