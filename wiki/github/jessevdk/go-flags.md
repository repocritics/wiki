# jessevdk/go-flags

> A reflection- and struct-tag-driven command line option parser for Go, in the GNU getopt tradition.

[GitHub repo](https://github.com/jessevdk/go-flags) ·
[Docs](https://pkg.go.dev/github.com/jessevdk/go-flags) ·
[License: BSD-3-Clause](https://github.com/jessevdk/go-flags/blob/main/LICENSE)

## Overview

go-flags declares command line options as struct fields annotated with tags,
then populates them via reflection at parse time. It positions itself as a
richer alternative to the standard library `flag` package: it understands
GNU-style long options (`--verbose`), short option bundling (`-aux`), `=`-joined
and space-separated argument forms (`-I/usr/include`, `-I=/usr/include`),
required options, defaults, environment-variable fallbacks, choice
restrictions, maps, slices, callbacks, and nested option groups.[^1]

The defining tradeoff is the reflection-and-tags model. A whole CLI surface can
be described in one struct with almost no imperative wiring, which is concise
and readable. The cost is that option semantics live inside opaque string tags
(`short:"v" long:"verbose" required:"true"`) that the compiler does not check:
a misspelled tag key, a type the parser cannot marshal, or a malformed default
surfaces at runtime, not at build time. Teams that value declarative brevity
like it; teams that want the type system to catch mistakes prefer builder-style
APIs.

The project is mature and effectively feature-complete: created in 2012, with
the last substantive push to `main` in mid-2024, moderate issue volume, and low
commit cadence.[^2] Read it as a stable, quietly maintained dependency rather
than an actively evolving one.

## Getting Started

```bash
go get github.com/jessevdk/go-flags
```

```go
package main

import (
	"fmt"
	"github.com/jessevdk/go-flags"
)

type Options struct {
	Verbose []bool `short:"v" long:"verbose" description:"Show verbose debug information"`
	Name    string `short:"n" long:"name" description:"A name" required:"true"`
	Animal  string `long:"animal" choice:"cat" choice:"dog" default:"cat"`
	Offset  uint   `long:"offset" default:"0" env:"OFFSET"`
}

func main() {
	var opts Options
	// Parse reads os.Args, prints a formatted --help, and returns on error.
	args, err := flags.Parse(&opts)
	if err != nil {
		return // flags already printed the error/usage for common cases.
	}
	fmt.Printf("name=%s animal=%s rest=%v\n", opts.Name, opts.Animal, args)
}
```

`flags.Parse` uses `os.Args`; `flags.ParseArgs(&opts, args)` takes an explicit
slice. For subcommands, embed fields whose type satisfies `Commander`.

## Architecture / How It Works

The core is a `Parser` built from a pointer to a struct. At construction and
parse time it walks the struct with `reflect`, reading field tags to build an
internal model of options and groups:

- **Options** come from tagged fields. Supported destination types include the
  Go primitives (`string`, the sized `int`/`uint` variants, `float`, `bool`),
  slices and maps of them, pointers, and `func(string)` callbacks invoked once
  per occurrence.[^1] A `[]bool` field is the idiom for repeatable flags (`-vvv`).
- **Groups and namespaces** come from embedded/nested structs tagged with
  `group` and optionally `namespace`, which prefix the contained long options —
  this is how large tools partition help output.
- **Commands** (subcommands) come from fields whose type implements `Commander`
  (an `Execute([]string) error` method). The parser dispatches to the matched
  command and can require one be given (`SubcommandsOptional` toggles this).
- **Help generation** is built in: the parser renders GNU-style aligned usage
  text and intercepts `-h`/`--help`, returning a `*flags.Error` of type
  `ErrHelp` that programs are expected to detect.

Parser behavior is tuned with an `Options` bitmask passed to `NewParser`
(`flags.HelpFlag`, `flags.PassDoubleDash`, `flags.IgnoreUnknown`, and others);
`flags.Default` bundles the common set (print errors, add `--help`, pass
everything after `--` through).

A distinguishing feature beyond argument parsing is the **`IniParser`**: the
same option struct can be serialized to and read from INI-style config files, so
a program can accept identical settings from flags, environment, and a config
file without a second schema.[^3] This overlaps with what many Go CLIs reach for
Viper to do. Custom types are handled by implementing `flags.Marshaler` /
`flags.Unmarshaler` or the standard `encoding.TextUnmarshaler` — the escape
hatch for anything the built-in type handling does not cover.

## Production Notes

- **Errors are runtime, not compile time.** A tag typo (`shrt:` for `short:`) is
  silently ignored; an unsupported destination type or a bad `default` errors or
  panics only when the parser runs. Cover the option struct with at least one
  parse test so these surface in CI, not in a user's terminal.
- **`ErrHelp` is not a failure.** `flags.Parse` returns a non-nil error when the
  user passes `--help`. Idiomatic code checks `flags.WroteHelp(err)` and exits
  zero; treating every non-nil error as a failure makes `--help` exit non-zero.
- **`required` interacts badly with `--help` and subcommands.** A required option
  can trigger a "required flag not specified" error before help is shown, and
  required fields on a parent group are enforced even when the user only wants a
  subcommand's usage. Scope required-ness to the command, not the top-level
  struct.
- **Versioning.** Go modules resolve it via the `v1.x` tags; there is no `/v2`
  module path. Pin a released tag rather than tracking `main`.
- **Maintenance posture.** With the last release batch and last `main` push both
  in 2024, expect slow response to issues and PRs. Reflection overhead is
  irrelevant on a CLI's cold parse path — the real cost is debuggability, not
  speed. It is dependable precisely because it changes rarely, but do not count
  on new features landing.

## When to Use / When Not

**Use when:**
- You want the whole CLI surface described declaratively in one struct with
  minimal imperative code.
- You need GNU/POSIX getopt semantics (short bundling, `--long=value`, `--`
  passthrough) that the stdlib `flag` package does not provide.
- You want env-var fallbacks, choices, defaults, and optional INI config wired
  from the same field definitions without extra libraries.

**Avoid when:**
- You are building a large, evolving multi-command tool with shell completion,
  man-page generation, and a big community of examples — cobra is the ecosystem
  default there.
- You want option definitions the compiler can verify; a builder/registration
  API keeps mistakes out of runtime.
- You need an actively developed dependency with fast maintainer turnaround.

## Alternatives

- spf13/cobra — the de facto framework for large Go CLIs (kubectl, hugo,
  gh-style tools); use it when you need subcommands, completion, and docs
  generation at scale.
- spf13/pflag — a drop-in GNU-style replacement for stdlib `flag`; use it when
  you only need better flag parsing without the struct-tag model (it underpins
  cobra).
- urfave/cli — imperative command/flag registration with a large user base; use
  it when you prefer explicit builders over reflection tags.
- alecthomas/kingpin — fluent builder API with strong validation; use it when
  you want type-safe definitions and rich terminal help.
- standard library `flag` — use it when your needs are minimal and a zero-
  dependency, non-GNU-style parser is acceptable.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1 | 2013-08-26 | First tagged release.[^4] |
| v1.4.0 | 2018-03-31 | Release published on GitHub.[^4] |
| v1.5.0 – v1.6.1 | 2024-06-15 | Release objects published together for existing tags; v1.6.1 is the current tag.[^4] |
| (main) | 2024-07-26 | Last push to the default branch as of this writing.[^2] |

Note: the v1.5.0–v1.6.1 release objects were all published on the same day; the
underlying tags predate them, so the dates reflect when releases were cut, not
when the code first landed.

## References

[^1]: Package README and feature list, jessevdk/go-flags. https://github.com/jessevdk/go-flags/blob/main/README.md
[^2]: Repository metadata via GitHub API (last push to `main` 2024-07-26; not archived). https://github.com/jessevdk/go-flags
[^3]: `IniParser` and INI config support, package documentation. https://pkg.go.dev/github.com/jessevdk/go-flags#IniParser
[^4]: Release and tag list, GitHub API `repos/jessevdk/go-flags/releases`. https://github.com/jessevdk/go-flags/releases

## Tags

go, golang, cli, command-line, argument-parser, flags, getopt, struct-tags, reflection, config, library
