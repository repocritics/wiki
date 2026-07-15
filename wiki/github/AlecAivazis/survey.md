# AlecAivazis/survey

> A Go library for interactive terminal prompts (input, select, multi-select, password, editor) with Windows and POSIX support — now archived and unmaintained.

[GitHub repo](https://github.com/AlecAivazis/survey) ·
[License: MIT](https://github.com/AlecAivazis/survey/blob/master/LICENSE)

## Overview

survey is a Go library for building interactive command-line prompts on
terminals that support ANSI escape sequences[^1]. It provides a small set of
ready-made prompt widgets — `Input`, `Multiline`, `Password`, `Confirm`,
`Select`, `MultiSelect`, and `Editor` — and a reflection-based mechanism that
writes user answers directly into the fields of a Go struct. For most of the
late-2010s it was the default choice for Go CLI authors who wanted a wizard-style
prompt without hand-rolling raw-mode terminal handling, and it accumulated a
broad dependency footprint across the Go tooling ecosystem (~4.1k stars, ~350
forks as of mid-2026).

The defining fact about survey today is that **it is archived and no longer
maintained**. In April 2024 the author closed the project, stating he could no
longer dedicate time to it, and pointed users toward charmbracelet/bubbletea as
a successor direction[^2]. The repository still builds and works, but it will
not receive security fixes, dependency bumps, or compatibility updates for newer
Go releases. Treat it as frozen infrastructure: fine for existing tools, a
liability for new ones that need a long support horizon.

The library's second defining trait is its "magic struct" API. Answers are
mapped onto struct fields by name (or by a `survey:"..."` tag), and survey
performs type conversion via reflection. This is convenient for simple forms and
opaque when something goes wrong — mismatched types, unexported fields, or
custom types all fail at runtime rather than compile time.

## Getting Started

```bash
go get github.com/AlecAivazis/survey/v2
```

```go
package main

import (
	"fmt"

	"github.com/AlecAivazis/survey/v2"
)

func main() {
	qs := []*survey.Question{
		{Name: "name", Prompt: &survey.Input{Message: "What is your name?"}, Validate: survey.Required},
		{Name: "color", Prompt: &survey.Select{
			Message: "Choose a color:",
			Options: []string{"red", "blue", "green"},
			Default: "red",
		}},
	}

	answers := struct {
		Name          string
		FavoriteColor string `survey:"color"` // tag maps question -> field
	}{}

	if err := survey.Ask(qs, &answers); err != nil {
		fmt.Println(err.Error())
		return
	}
	fmt.Printf("%s chose %s.\n", answers.Name, answers.FavoriteColor)
}
```

Use `survey.AskOne(prompt, &target, opts...)` for a single value and
`survey.Ask(questions, &struct, opts...)` for a batch. Behavior is tuned either
through fields on the prompt struct or through `AskOpt` functions such as
`survey.WithValidator`, `survey.WithPageSize`, and `survey.WithFilter`.

## Architecture / How It Works

survey is split into a public prompt API and an internal `terminal` package that
handles the low-level work. When a prompt runs, the terminal package puts the TTY
into raw mode, reads input rune-by-rune, interprets control codes and cursor
movement, and repaints the prompt on each keystroke. Rendering is
template-driven: each prompt type carries a `text/template` string with color
helper functions (backed by mgutz/ansi) so icons, focus markers, and error text
can be re-themed via `WithIcons`.

The struct-binding layer is the other half. `survey.Ask` walks the answer struct
with reflection, matches each question `Name` to a field (case-insensitively, or
via the `survey` tag), converts the answer to the field's type, and assigns it.
Custom types can intercept this by implementing the `Settable` interface
(`WriteAnswer(field string, value interface{}) error`), which is also how nested
or non-trivial answers are supported.

Two design choices dominate everything downstream:

1. **Hard TTY dependency.** survey assumes an interactive terminal that
   understands ANSI escape sequences. It manipulates the cursor and reads raw
   bytes, so piped stdin or redirected stdout is explicitly unsupported and will
   generally corrupt output or hang[^3].
2. **Control-code capture.** While a prompt is active, survey reconfigures the
   terminal so keys like Ctrl-C arrive as ordinary input bytes rather than
   signals. It converts a `^C` byte into a `terminal.InterruptErr` returned from
   `Ask`/`AskOne` — the process does **not** exit on its own[^4].

The `/v2` module path (introduced with the version 2 redesign) is a frequent
source of confusion, because a large body of older tutorials and answers still
import the unversioned v1 path with an incompatible API.

## Production Notes

**Non-TTY environments break it.** This is the single most common failure. In CI,
when stdin is piped, or when output is captured, survey misbehaves — the fix is
to detect non-interactive contexts (e.g. check `isatty`) and provide a flag-based
code path instead of prompting. Do not assume prompts are safe to reach in
automation.

**You must handle `terminal.InterruptErr` yourself.** If you ignore the error
from `Ask`/`AskOne`, a user pressing Ctrl-C will not terminate the program the
way they expect. Idiomatic usage checks `errors.Is(err, terminal.InterruptErr)`
and exits explicitly.

**Archived means frozen dependencies.** Because the repo is unmaintained, its
transitive dependencies are pinned to their 2024 state. New Go releases are not
being validated against it, and any CVE in a dependency will not be patched
upstream. If you keep survey, vendor it, run your own dependency audit, and be
prepared to fork for compatibility.

**The `Editor` prompt shells out.** It launches `$VISUAL`/`$EDITOR` (falling back
to vim on Unix, notepad on Windows) against a temp file. That is a process-spawn
and a filesystem write on every use — relevant for sandboxed or restricted
environments, and it inherits whatever the user's editor environment is.

**Windows support hinges on ANSI.** survey advertises Windows compatibility, but
that depends on the console interpreting virtual-terminal sequences. Modern
Windows Terminal is fine; older `cmd.exe`/ConHost configurations historically
needed VT processing enabled.

**Testing requires a virtual terminal.** Because `os.Stdout` in `go test` is not
a TTY, prompt tests need go-expect together with a vt10x pseudo-terminal to
interpret cursor and escape sequences[^5]. Expect prompt-level tests to be
heavier than ordinary unit tests.

## When to Use / When Not

**Use when:**
- You already depend on it in a shipping tool and it works — there is no urgent
  reason to churn.
- You need a small, familiar prompt set (input/select/confirm/password) and can
  accept a frozen dependency.
- You want struct-based answer binding rather than wiring each value manually.

**Avoid when:**
- You are starting a new project and want an actively maintained library with
  security updates.
- You need to run in non-interactive or piped contexts, or want a full-screen TUI
  rather than sequential prompts.
- You want compile-time safety over the reflection-based struct mapping.

## Alternatives

- charmbracelet/huh — the closest actively-maintained replacement: form and
  prompt widgets with struct/value binding, built on Bubble Tea.
- charmbracelet/bubbletea — use when you need a full TUI framework (Elm-style
  model/update/view) rather than one-shot prompts; the maintainer's own pointer.
- manifoldco/promptui — use when you want a lighter, prompt-only library with a
  similar select/input scope.
- c-bata/go-prompt — use when interactive autocompletion and a REPL-style input
  line are the priority.
- erikgeiser/promptkit — use when you want small, composable Bubble Tea-based
  prompt components without adopting a whole framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2016–2018 | Initial library; unversioned `github.com/AlecAivazis/survey` import path. |
| 2.0 | 2019 | Version 2 redesign; module path moved to `/v2`, `AskOpt`-based configuration. |
| 2.3.x | 2021–2023 | Feature additions: option descriptions, `KeepFilter`, `WithRemoveSelectAll`/`SelectNone`, suggestion completion. |
| — | 2024-04-07 | Final commit; repository archived and marked unmaintained, recommending charmbracelet/bubbletea[^2]. |

## References

[^1]: survey README — "A library for building interactive and accessible prompts on terminals supporting ANSI escape sequences." https://github.com/AlecAivazis/survey
[^2]: Archival notice in README — "This project is no longer maintained. For an alternative, please check out: charmbracelet/bubbletea." https://github.com/AlecAivazis/survey
[^3]: survey FAQ — "reading from piped stdin or writing to piped stdout is not supported." https://github.com/AlecAivazis/survey/pull/337
[^4]: survey FAQ — Ctrl-C handling returns `terminal.InterruptErr` rather than terminating the process. https://github.com/AlecAivazis/survey
[^5]: survey Testing section — go-expect + vt10x virtual console for prompt tests. https://github.com/Netflix/go-expect

## Tags

go, cli, terminal, interactive-prompt, tui, command-line, ansi, unmaintained, library, developer-tools
