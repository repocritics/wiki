# urfave/cli

> A declarative Go package for building command-line tools, with commands, flags, and shell completion and no dependencies beyond the standard library.

[GitHub repo](https://github.com/urfave/cli) ·
[Official website](https://cli.urfave.org) ·
[License: MIT](https://github.com/urfave/cli/blob/main/LICENSE)

## Overview

urfave/cli is one of the two long-standing CLI frameworks in the Go ecosystem
(the other being spf13/cobra). It began life as `codegangsta/cli` by Jeremy
Saenz in 2013 and was later moved under the `urfave` organization[^1]. The design
goal is a declarative API: you describe a `Command` tree as a struct literal —
name, usage, flags, subcommands, and an action function — and the library handles
parsing, help text, and dispatch. There is no code generation and no runtime
dependency outside the Go standard library, which keeps it easy to vendor and audit.

The project's defining tension is its two live major versions. v2 (`github.com/urfave/cli/v2`)
is the version most existing code and tutorials target, and remains widely
deployed. v3 (`github.com/urfave/cli/v3`) is the current line the README and docs
point at; it threads `context.Context` through actions, unifies the old `App` and
`Command` types into a single `Command`, and changes how flags are read. The two
versions are separate import paths and do not interoperate within one binary, so
"which urfave/cli" is a real decision, not a version bump[^2].

Compared with cobra, urfave/cli is lighter and more literal: less machinery, no
generator, no separate pflag dependency. That is a feature if you want a small
declarative surface and a liability if you need cobra's POSIX/GNU flag semantics
or its large plugin ecosystem. With ~24k stars and steady recent commits, it is
actively maintained, though the maintainers note explicitly that the project is
run by unpaid volunteers.

## Getting Started

```bash
go get github.com/urfave/cli/v3@latest
```

```go
// main.go — urfave/cli v3
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/urfave/cli/v3"
)

func main() {
	cmd := &cli.Command{
		Name:  "greet",
		Usage: "say a greeting",
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:  "name",
				Value: "world",
				Usage: "who to greet",
			},
		},
		Action: func(ctx context.Context, cmd *cli.Command) error {
			fmt.Printf("Hello, %s\n", cmd.String("name"))
			return nil
		},
	}

	if err := cmd.Run(context.Background(), os.Args); err != nil {
		log.Fatal(err)
	}
}
```

In v2 the entry point is `cli.App`, the action signature is `func(c *cli.Context) error`,
and flags are read via `c.String("name")`. Porting between the two is mostly
mechanical but touches every action.

## Architecture / How It Works

The core type is `Command`. A root command holds `Flags`, an `Action`, and a
`Commands` slice of subcommands; each subcommand is itself a `Command`, so the CLI
is a tree of the same struct. `Run` walks `os.Args`, matches the argument path to a
subcommand, parses flags at each level, and invokes the deepest matched command's
`Action`. Alias and prefix matching let `app c` resolve to `app commit` when
unambiguous.

Flags are interface values (`cli.Flag`) with concrete types per value kind:
`StringFlag`, `IntFlag`, `BoolFlag`, `DurationFlag`, `TimestampFlag`, and slice
variants like `StringSliceFlag`. Each flag can source its value from, in order,
the command line, environment variables (`Sources: cli.EnvVars("PORT")` in v3),
and plain-text files. Structured-file sources (YAML/TOML/JSON) were split out of
core into the separate `urfave/cli-altsrc` module, and man/Markdown documentation
generation lives in `urfave/cli-docs`[^3] — a deliberate move to keep the core
dependency-free and small.

Parsing is urfave/cli's own, not `flag` or `pflag`. It supports compound short
flags (`-abc` == `-a -b -c`) and its own conventions for long flags. This is the
main behavioral difference from cobra, which delegates to pflag and follows
GNU-style `--flag=value`. Shell completion (`bash`, `zsh`, `fish`, `powershell`)
is generated from the command tree at runtime rather than emitted as static
scripts.

The v2→v3 rewrite is the significant architectural event. v3 removes the
`App`/`Command` split, adds `context.Context` as the first action argument (so
cancellation and request-scoped values flow through the command tree), and reads
flag values off the `*cli.Command` passed to the action instead of a separate
`*cli.Context`. These are source-incompatible changes, which is why v3 spent
years in beta before stabilizing.

## Production Notes

**Pick a major version deliberately.** v2 and v3 are different import paths.
Mixing them (e.g. a library on v2 and your app on v3) is fine at the module level
but you cannot pass one's types to the other. Most third-party tutorials, Stack
Overflow answers, and existing internal tooling still assume v2's `App` + `*cli.Context`
API, so v3 adopters should expect to translate examples by hand.

**Flag parsing is not POSIX/GNU.** If your users or scripts expect strict
`getopt`-style behavior, interspersed flags and arguments, or `--` handling
identical to standard Unix tools, verify urfave/cli's parser matches — historically
its ordering and interspersing rules have differed from pflag/cobra and have been
a recurring source of issues. Test the exact invocations your users will type.

**No built-in config-file layering in core.** Reading flags from YAML/TOML/JSON
requires the `urfave/cli-altsrc` module; it is not in the box. Similarly,
`man`/Markdown doc generation is in `urfave/cli-docs`. Budget for these as extra
modules rather than assuming the core package covers them.

**Global vs local flags.** Flags defined on the root are available to
subcommands, but the lookup semantics (which level's value wins, how a flag set on
the parent is read from a child action) have edge cases. When a persistent/global
flag doesn't appear where expected, check whether you're reading it from the right
command in the tree.

**Volunteer maintenance.** The project is stable and moves steadily, but the
maintainers state it is run by unpaid volunteers and route support to GitHub
Discussions Q&A rather than guaranteeing issue response times. Treat feature
requests accordingly.

## When to Use / When Not

**Use when:**
- You want a small, declarative, dependency-free CLI defined as a struct tree.
- You value a minimal surface and easy vendoring/auditing over a large ecosystem.
- Your flag needs are simple types, slices, durations, and env/file sourcing.
- You're starting fresh and can adopt v3's `context.Context`-threaded API.

**Avoid when:**
- You need strict POSIX/GNU flag semantics — cobra + pflag is the safer default.
- You want a generator, a plugin ecosystem, or alignment with the tooling used by
  Kubernetes/Docker/Hugo/GitHub CLI (all cobra).
- You need first-class config-file layering or doc generation without pulling in
  additional urfave modules.

## Alternatives

- spf13/cobra — the dominant Go CLI framework (kubectl, docker, hugo, gh); use it when you need POSIX flags, a generator, and ecosystem alignment.
- spf13/pflag — drop-in POSIX/GNU replacement for stdlib `flag`; use it when you only need flag parsing, not a command framework.
- alecthomas/kong — struct-tag-driven CLI parser; use it when you prefer declaring the CLI as annotated Go structs over builder code.
- alecthomas/kingpin — fluent-builder CLI library; use it for a mature builder-style API (now largely superseded by kong).
- jessevdk/go-flags — struct-tag `getopt`-style parser; use it when you want GNU-compatible parsing driven entirely by struct tags.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07 | Created as `codegangsta/cli` by Jeremy Saenz[^1]. |
| v1.x | 2014–2016 | Stable v1 line; later moved under the `urfave` org. |
| v2.0.0 | 2019-08 | New import path `urfave/cli/v2`; API cleanup, `altsrc` reworked[^2]. |
| v3 (beta) | 2021–2024 | Long beta: `context.Context` in actions, `App`/`Command` merged. |
| v3.0.0 | 2025 | v3 line stabilized; current default import path `urfave/cli/v3`[^2]. |

## References

[^1]: urfave/cli repository, created 2013-07-13; originally `codegangsta/cli`. https://github.com/urfave/cli
[^2]: urfave/cli releases and version migration notes (v2 vs v3 import paths). https://github.com/urfave/cli/releases
[^3]: Companion modules referenced in the README: `urfave/cli-altsrc` (structured config sources) and `urfave/cli-docs` (man/Markdown generation). https://github.com/urfave/cli-altsrc
[^4]: Hosted documentation. https://cli.urfave.org

## Tags

go, cli, command-line, framework, flag-parsing, shell-completion, golang-library, terminal, argument-parsing, declarative
