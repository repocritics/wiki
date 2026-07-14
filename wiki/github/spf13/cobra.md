# spf13/cobra

> The command/flag framework that most large Go CLIs are built on — kubectl, gh, helm, hugo, and docker all sit on top of it.

[GitHub repo](https://github.com/spf13/cobra) ·
[Official website](https://cobra.dev) ·
[License: Apache-2.0](https://github.com/spf13/cobra/blob/main/LICENSE.txt)

## Overview

Cobra is a Go library for building subcommand-based command-line tools of the
`app verb noun --flag` shape (`git clone`, `kubectl get pods`). It was started
in 2013 by Steve Francia (spf13), the same author as Hugo and Viper, and grew
out of the CLI needs of those projects[^1]. Over a decade it became the de
facto standard for non-trivial Go CLIs: Kubernetes, GitHub CLI, Helm,
containerd, etcd, Istio, Rclone, and Docker/Moby all build their command trees
with it[^2].

Structurally Cobra is a tree of `*cobra.Command` values plus a flag layer. It
does not parse flags itself — that job belongs to spf13/pflag, a fork of the
standard library `flag` package that adds POSIX/GNU-style short and long
flags[^3]. Cobra optionally pairs with Viper for configuration binding. This
triad (cobra + pflag + viper) is powerful but means adopting Cobra pulls in an
opinionated ecosystem rather than a single small dependency.

The defining tension is boilerplate versus control. Cobra gives you generated
help, shell completion, man pages, and suggestion ("did you mean") for free,
but it asks you to wire commands imperatively — usually via package-level vars
and `init()` functions — which produces verbose, globally-stateful code that is
awkward to unit-test. Projects that want declarative struct-tag CLIs
(urfave/cli, kong) trade some of Cobra's features for far less wiring.

## Getting Started

```bash
go get -u github.com/spf13/cobra@latest
# optional scaffolding generator (separate module since 2022):
go install github.com/spf13/cobra-cli@latest
```

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

func main() {
	var verbose bool

	root := &cobra.Command{
		Use:   "greet [name]",
		Short: "Print a greeting",
		Args:  cobra.ExactArgs(1), // without this, extra args are silently accepted
		RunE: func(cmd *cobra.Command, args []string) error {
			if verbose {
				fmt.Fprintln(os.Stderr, "verbose on")
			}
			fmt.Printf("hello, %s\n", args[0])
			return nil
		},
	}
	root.Flags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")

	if err := root.Execute(); err != nil {
		os.Exit(1) // Execute already printed the error
	}
}
```

## Architecture / How It Works

A Cobra program is a directed tree of `Command` structs. You build the tree
with `parent.AddCommand(child)` and start execution with `rootCmd.Execute()`,
which locates the target command by walking `os.Args`, parses flags via pflag,
runs the args validator, fires the pre-run hooks, and finally calls `Run`/`RunE`.

Flags come in two scopes: `Flags()` are local to a command, and
`PersistentFlags()` cascade to all descendants. Persistent flags are the usual
way to implement global options like `--config` or `-v`. The hook chain
(`PersistentPreRun`, `PreRun`, `Run`, `PostRun`, `PersistentPostRun`) runs
around execution — but only the *nearest* persistent hook in the ancestry runs
by default, a subtlety covered under Production Notes.

Two design choices shape everything downstream. First, most real programs
register commands and bind flags in `init()` with package-level variables, so
the command tree is effectively global mutable state. Second, `Execute()` does
its own error printing and, by convention, the tool decides exit codes from the
returned error rather than Cobra choosing for you.

Shell completion is generated programmatically, not as a static script dump:
Cobra injects a hidden `__complete` command that the emitted bash/zsh/fish/
PowerShell scripts call back into at runtime, so completions can reflect live
state (available pods, remote branches)[^4]. Man-page and Markdown docs
generation live in the `doc` subpackage.

The scaffolding generator was carved out of the main repository into
spf13/cobra-cli around 2022[^5], so the library and the `cobra-cli` binary now
version independently — a frequent source of confusion for anyone following
older tutorials that ran a bundled `cobra` command.

## Production Notes

- **`Args` defaults to permissive.** If you do not set an `Args` validator
  (e.g. `cobra.ExactArgs`, `cobra.NoArgs`), extra positional arguments are
  accepted and silently ignored. This is the single most common Cobra bug.
- **Persistent pre-run hooks do not stack.** A child's `PersistentPreRunE`
  *replaces* an ancestor's rather than running in addition to it. Shared setup
  in a parent hook silently stops running once a child defines its own; you
  must call the parent hook explicitly or centralize setup on the root.
- **`Run` vs `RunE`.** Use `RunE` and return errors. Pair it with
  `SilenceUsage: true` and `SilenceErrors: true` on the root, or Cobra prints
  the full usage text on every runtime error, which reads like a parse failure.
- **Global state fights tests.** Because commands and flags are usually
  package-level, tests that call `Execute()` twice leak flag values between
  runs. Rebuild the command in a factory function per test, or reset flags
  manually. Cobra provides `cmd.SetArgs`/`SetOut`/`SetErr` for capturing I/O.
- **Windows GUI launch.** Cobra's mousetrap behavior makes a program launched
  by double-click from Explorer print a message and wait for a keypress. It is
  intended, disable via `cobra.MousetrapHelpText = ""` if your tool is meant to
  run head-less on Windows[^6].
- **Deep trees and `init()` ordering.** Very large command trees built across
  many `init()` functions have nondeterministic construction order and slow
  startup from eager registration; some big CLIs lazy-build subtrees.
- **Stability.** Cobra is mature and still on the 1.x line — there has been no
  breaking v2. Upgrades are usually low-drama, but the completion subsystem was
  reworked over the 1.0–1.1 window, so tools carrying hand-written completion
  scripts from the 0.x era should migrate to the generated `completion` command.

## When to Use / When Not

**Use when:**
- You are building a multi-command CLI (`app sub subsub`) and want generated
  help, suggestions, shell completion, and man pages without writing them.
- You want the same foundations as kubectl/gh/helm for familiarity and hiring.
- You plan to bind config from files/env via Viper alongside flags.

**Avoid when:**
- Your tool has one or two commands — stdlib `flag` or urfave/cli is far less code.
- You want declarative, struct-tag-driven command definitions and easy testing.
- You object to package-level global command state or the pflag/viper pull-in.

## Alternatives

- urfave/cli — map/struct-based CLI framework, much less boilerplate; the usual choice when Cobra feels heavy.
- alecthomas/kong — struct-tag, declarative command/flag definitions; use when you want type-driven CLIs that test cleanly.
- spf13/pflag — the flag layer alone; use when you need POSIX flags but not a command tree.
- google/subcommands — minimal subcommand dispatch; use for small tools that only need `tool verb`.
- charmbracelet/fang — an opinionated wrapper over Cobra for styled output; use when you like Cobra but want nicer defaults.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2013-09 | Started by spf13 alongside Hugo/Viper[^1]. |
| pre-1.0 (0.x) | 2015–2019 | Widely adopted (kubectl, hugo, docker) before a tagged 1.0. |
| 1.0.0 | 2020 | First stable tag; Go-driven programmatic shell completion[^4]. |
| cobra-cli split | 2022 | Generator moved to spf13/cobra-cli, versioned separately[^5]. |
| 1.x (ongoing) | 2020–present | Actively maintained on the 1.x line; no breaking v2 to date. |

## References

[^1]: Cobra project site and origin. https://cobra.dev
[^2]: "Projects using Cobra." https://github.com/spf13/cobra/blob/main/site/content/projects_using_cobra.md
[^3]: spf13/pflag — drop-in POSIX/GNU flag replacement for the stdlib `flag` package. https://github.com/spf13/pflag
[^4]: Cobra shell completions documentation. https://github.com/spf13/cobra/blob/main/site/content/completions/_index.md
[^5]: spf13/cobra-cli — the extracted generator. https://github.com/spf13/cobra-cli
[^6]: Cobra API reference (`MousetrapHelpText`, `Command`, `Args`). https://pkg.go.dev/github.com/spf13/cobra

## Tags

go, golang, cli, command-line, cli-framework, subcommands, posix-flags, pflag, developer-tools, library
