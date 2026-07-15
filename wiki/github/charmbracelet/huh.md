# charmbracelet/huh

> A Go library for building interactive terminal forms and prompts, built on Bubble Tea.

[GitHub repo](https://github.com/charmbracelet/huh) ·
[Go docs](https://pkg.go.dev/charm.land/huh/v2) ·
[License: MIT](https://github.com/charmbracelet/huh/blob/main/LICENSE)

## Overview

`huh?` is Charm's library for collecting structured input in the terminal:
single- and multi-line text, single- and multi-select lists, and yes/no
confirmations, grouped into multi-page forms with validation. It was first
released in December 2023[^1] and is the form/prompt layer of the broader Charm
stack (Bubble Tea, Lip Gloss, Bubbles). It is explicitly inspired by Alec
Aivazis's `survey`[^2], which it has largely displaced as the default choice for
new Go CLIs.

The library has two personalities that share one API. In *standalone* mode you
build a `huh.Form`, call `form.Run()`, and it takes over the terminal, runs an
internal Bubble Tea program, and blocks until the user finishes — the fast path
for a wizard or an installer. Because a `huh.Form` is itself a `tea.Model`, the
same form can instead be *embedded* inside a larger Bubble Tea application, where
you drive its `Init`/`Update`/`View` yourself. This dual nature is the defining
design decision: it keeps the simple case a few lines long while letting complex
TUIs reuse the exact same field code[^3].

The main tension is that huh inherits the entire Charm rendering model. That
buys polished, themeable output and accessibility, but it also means huh assumes
an interactive TTY, adapts colors to detected terminal background, and carries
the styling-heavy dependency footprint of Lip Gloss and Bubble Tea — none of
which is free in a non-interactive or resource-constrained context.

## Getting Started

```bash
go get charm.land/huh/v2@latest
```

```go
import "charm.land/huh/v2"

var name, topping string

form := huh.NewForm(
	huh.NewGroup(
		huh.NewInput().
			Title("What's your name?").
			Value(&name).
			Validate(func(s string) error {
				if s == "" {
					return fmt.Errorf("name is required")
				}
				return nil
			}),
		huh.NewSelect[string]().
			Title("Pick a topping").
			Options(huh.NewOptions("Lettuce", "Tomato", "Cheese")...).
			Value(&topping),
	),
)

if err := form.Run(); err != nil { // takes over the terminal, blocks
	log.Fatal(err)
}
fmt.Printf("%s ordered %s\n", name, topping)
```

Values are bound by pointer: each field writes into the variable you pass to
`Value(&x)` as the user commits input. `Select`/`MultiSelect` are generic over
the option type, so `NewSelect[int]()` stores integers, `NewSelect[string]()`
stores strings, and so on.

## Architecture / How It Works

huh is a thin, opinionated composition layer over Bubble Tea's Elm architecture
(immutable model, `Update(msg) (model, cmd)`, `View() string`). The pieces nest:
a `Form` owns ordered `Group`s (rendered one at a time, like pages); each Group
owns `Field`s; each Field (`Input`, `Text`, `Select`, `MultiSelect`, `Confirm`,
plus `Note` and a file picker) is itself a `tea.Model` that renders with Lip
Gloss and, for text entry, wraps the `bubbles` textinput/textarea components.

Because every level implements `tea.Model`, `form.Run()` is essentially sugar
that constructs a `tea.Program`, runs it, and returns once the form reaches
`StateCompleted`. Embedded usage skips that wrapper and hands messages to
`form.Update` from your own program. State is read back either through the bound
pointers or through keyed getters (`form.GetString("key")`, `GetInt`, etc.) when
you set `.Key(...)` on fields.

Dynamic forms are handled by `*Func` variants of the property setters
(`TitleFunc`, `OptionsFunc`, `ValidateFunc`, …). Each takes a closure plus a
`binding any`; huh recomputes the property only when the bound value changes and
caches the result otherwise[^4]. This is how a field's options can depend on an
earlier answer (country → state) without recomputing — or re-hitting an API — on
every keypress. Passing the wrong binding is the classic footgun here.

Theming is a first-class abstraction built on Lip Gloss styles. Five themes ship
(`Charm`, `Dracula`, `Catppuccin`, `Base16`, `Default`), and a custom `*Theme`
can restyle every element. A separate accessible renderer, enabled with
`form.WithAccessible(true)`, drops the TUI entirely in favor of plain sequential
prompts for screen readers.

## Production Notes

**huh requires a real TTY.** `form.Run()` expects an interactive terminal; in CI,
under a pipe, or in a non-interactive container it will fail or misbehave rather
than degrade gracefully. Gate interactive prompts behind a TTY check and provide
flag-based fallbacks for automation. The accessible renderer
(`WithAccessible(true)`, typically wired to an env var) is also the more
robust path for constrained environments.

**The v1 → v2 migration is a module-path break.** v2.0.0 (March 2026) moved the
import path to `charm.land/huh/v2`; older code imports `github.com/charmbracelet/huh`[^5].
This is a hard, explicit upgrade — you change every import and re-`go get` — not
a transparent minor bump. Pin the major version deliberately and expect the two
lines of the ecosystem (v1 vs v2) to coexist in the wild for a while.

**Rendering depends on terminal detection.** Colors and adaptive styling come
from Lip Gloss's background-color detection, which can guess wrong over SSH, in
tmux, in some CI log viewers, or in terminals that don't answer the query —
producing washed-out or invisible text. Setting an explicit theme and, where
needed, forcing color profiles is the usual mitigation. Rendering fidelity on
older Windows consoles is weaker than on Unix terminals, as with the whole Charm
stack.

**Value binding is pointer-lifetime sensitive.** The variables passed to
`Value(&x)` must outlive the form run; binding to a loop variable or a value that
goes out of scope is a common source of "my answers disappeared" bugs. The form
is also not designed to be mutated from other goroutines while `Run()` is in
progress — treat it as single-threaded during interaction and read results only
after it completes.

**Dependency weight.** Pulling in huh pulls in Bubble Tea, Lip Gloss, and
Bubbles. For a tool whose only need is a one-line yes/no prompt, that is a large
transitive tree; a smaller prompt library may be the better fit. huh earns its
weight when you have several fields, want validation and theming, or already
depend on the Charm stack.

## When to Use / When Not

**Use when:**
- You need multi-field, multi-page forms with validation in a Go CLI.
- You already build with Bubble Tea and want form input that composes natively.
- You want polished, themeable prompts and built-in screen-reader support.
- Option values need to be typed (generic `Select[T]`) rather than stringly.

**Avoid when:**
- The program must run non-interactively (CI, pipes, cron) as its primary mode.
- You only need one trivial prompt and want a minimal dependency footprint.
- You need broad legacy-Windows-console fidelity or a non-TTY renderer.
- You want a stable import path today and can't absorb the v1→v2 path change.

## Alternatives

- AlecAivazis/survey — the predecessor huh was inspired by; simpler, widely used, but far less actively developed. Reach for it only to match an existing codebase.
- charmbracelet/bubbletea — use directly when you need a full TUI, not just form input; huh sits on top of it.
- manifoldco/promptui — lighter single-prompt library; use when you want minimal dependencies and don't need multi-page forms.
- pterm/pterm — general terminal-UI toolkit with interactive prompts among many other widgets; use when prompts are one small part of a broader console UI.
- erikgeiser/promptkit — small, focused selection/text-input prompts; a middle ground between promptui and the full Charm stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2023-12-12 | Initial release — core fields, standalone + Bubble Tea modes[^1]. |
| v0.3.0 | 2024-01-25 | Early API expansion. |
| v0.5.0 | 2024-07-09 | Dynamic-form `*Func` bindings maturing. |
| v0.6.0 | 2024-09-06 | Continued field/theme work. |
| v0.7.0 | 2025-04-16 | — |
| v0.8.0 | 2025-10-14 | Last of the v0 line. |
| v1.0.0 | 2025–2026 | First stable major (tag). |
| v2.0.0 | 2026-03-09 | Module path moved to `charm.land/huh/v2`[^5]. |

## References

[^1]: huh releases — v0.1.0 published 2023-12-12. https://github.com/charmbracelet/huh/releases
[^2]: `survey` by Alec Aivazis, cited as huh's inspiration in the project README. https://github.com/AlecAivazis/survey
[^3]: huh README, "What about Bubble Tea?" — a `huh.Form` is a `tea.Model`. https://github.com/charmbracelet/huh#what-about-bubble-tea
[^4]: huh README, "Dynamic Forms" — `TitleFunc`/`OptionsFunc` with a `binding any` and automatic caching. https://github.com/charmbracelet/huh#dynamic-forms
[^5]: Go package path for v2. https://pkg.go.dev/charm.land/huh/v2

## Tags

go, tui, terminal, forms, prompt, cli, bubbletea, charm, interactive, accessibility
