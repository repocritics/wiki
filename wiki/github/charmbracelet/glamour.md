# charmbracelet/glamour

> Stylesheet-based Markdown rendering for CLI apps — turns Markdown into styled ANSI terminal output.

[GitHub repo](https://github.com/charmbracelet/glamour) ·
[License: MIT](https://github.com/charmbracelet/glamour/blob/main/LICENSE)

## Overview

Glamour is a Go library that renders Markdown documents to ANSI-styled text for
display in a terminal. It is the rendering engine behind Charm's own `glow`
viewer and is embedded in a number of widely-used CLI tools, including the
GitHub CLI (`gh`), the GitLab CLI, and the Gitea CLI, which use it to display
issue and PR bodies with headings, lists, tables, and syntax-highlighted code
blocks[^1]. First published in December 2019[^2], it is one of the mature pieces
of the Charm ecosystem alongside Bubble Tea and Lip Gloss, and as of 2026 sits
around 3.6k stars with steady maintenance rather than rapid feature churn.

The defining design choice is that the renderer is *pure*: the same Markdown
input plus the same stylesheet always produces byte-identical output, and the
renderer deliberately has no knowledge of the terminal it will be printed to.
That purity is what makes Glamour easy to test and embed, but it pushes two
concerns — color downsampling and terminal width detection — onto the caller.
This is the single most common source of surprise for people wiring Glamour into
a real app: it will happily emit 24-bit truecolor escape codes into a terminal
that cannot display them unless you explicitly downsample first (see Production
Notes).

Styling is data, not code. A "style" is a JSON document describing how each
Markdown element (headings, emphasis, code, block quotes, list bullets, table
borders) should be colored and spaced, so themes can be swapped, shipped as
files, or selected at runtime via the `GLAMOUR_STYLE` environment variable
without recompiling.

## Getting Started

```bash
go get charm.land/glamour/v2
```

```go
package main

import (
	"fmt"

	"charm.land/glamour/v2"
)

func main() {
	in := "# Hello World\n\nThis is *rendered* Markdown.\n"

	// One-shot render with a named built-in style.
	out, err := glamour.Render(in, "dark")
	if err != nil {
		panic(err)
	}
	fmt.Print(out)
}
```

For control over width and options, build a renderer explicitly:

```go
r, _ := glamour.NewTermRenderer(
	glamour.WithAutoStyle(),   // pick light/dark from terminal background
	glamour.WithWordWrap(80),  // hard-wrap column; default is 80
)
out, _ := r.Render(in)
fmt.Print(out)
```

## Architecture / How It Works

Glamour parses Markdown with the goldmark parser[^3], a CommonMark-compliant Go
library, and walks the resulting AST with a custom ANSI renderer instead of
goldmark's default HTML renderer. Each node type maps to a style block pulled
from the active JSON stylesheet, which is applied as ANSI escape sequences.
Word-wrapping and indentation are handled by Charm's reflow primitives, and code
blocks are syntax-highlighted through Chroma[^4], so the set of supported
languages and highlight themes is inherited from that project.

Styles ship as embedded JSON assets (`dark`, `light`, `notty`, `ascii`,
`dracula`, `pink`, and others). `notty` strips all escape sequences for plain
output, and `ascii` avoids Unicode box-drawing characters — both useful for
piping into files or non-interactive environments. A custom style is just a JSON
file; point `GLAMOUR_STYLE` at a path or a built-in name and call
`RenderWithEnvironmentConfig`, or pass `WithEnvironmentConfig()` to a custom
renderer.

The v2 line moved the import path to the `charm.land/glamour/v2` vanity module
(the source still lives at `github.com/charmbracelet/glamour`). Because Go
encodes the major version in the import path, v2 is a hard, opt-in upgrade rather
than a transparent `go get -u`.

## Production Notes

- **You must downsample colors yourself.** The renderer emits whatever colors the
  stylesheet specifies and does not inspect terminal capabilities. On a terminal
  that only supports 256 or 16 colors, truecolor output will render wrong.
  Charm's guidance is to pass Glamour's output through Lip Gloss (which does
  detect capabilities) before printing[^1]. This trips up nearly every first
  integration.
- **Width detection is the caller's job.** `WithWordWrap` takes a fixed column
  count; it does not query the TTY. If you want output to fill the current
  terminal you must read the width (e.g. via `golang.org/x/term`) and pass it in,
  and re-render on resize in a TUI.
- **Auto-styling reads the terminal background.** `WithAutoStyle()` /
  `WithStandardStyle` decisions depend on background-color detection, which is
  unreliable over some SSH sessions, inside tmux, or in CI. Pin an explicit style
  in non-interactive contexts to avoid unreadable low-contrast output.
- **Rendering is not free for large documents.** Because it parses to a full AST
  and reflows, rendering very large Markdown files on every keystroke in a TUI
  can be noticeable; cache rendered output and only re-render on content or width
  changes.
- **Not a security boundary.** Glamour renders untrusted Markdown for display,
  but escape-sequence handling in the *terminal* is out of scope; treat rendering
  arbitrary remote Markdown with the same caution as any terminal output.

## When to Use / When Not

**Use when:**
- You are building a Go CLI or Bubble Tea TUI and need to show Markdown (docs,
  README previews, issue/PR bodies, LLM output) with readable styling.
- You want themeable output driven by JSON stylesheets and environment config.
- You want deterministic, testable rendering you can snapshot in unit tests.

**Avoid when:**
- You are not in Go, or not targeting a terminal — this is ANSI-terminal output,
  not HTML.
- You need pixel-accurate layout, embedded images, or a full document viewer;
  reach for a browser or a dedicated TUI markdown viewer with image support.
- You only need to parse Markdown or emit HTML — use the underlying parser
  directly and skip the ANSI layer.

## Alternatives

- yuin/goldmark — use when you need to parse Markdown or emit HTML/custom output and don't need terminal styling (it's the parser Glamour builds on).
- MichaelMure/go-term-markdown — use when you want an alternative terminal Markdown renderer with inline image support.
- charmbracelet/lipgloss — use when you want to style terminal layout and text directly rather than render whole Markdown documents.
- charmbracelet/glow — use when you want a finished CLI/TUI Markdown reader instead of a library to embed.
- alecthomas/chroma — use when you only need syntax highlighting of code, not full Markdown rendering.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-12-18 | Repository created; ANSI Markdown renderer for Charm tools[^2]. |
| v0.x | 2020–2025 | Long v0 series; JSON stylesheets, `GLAMOUR_STYLE`, Chroma highlighting, adopted by `glow`, `gh`, and GitLab/Gitea CLIs[^1]. |
| v2 | 2026 | Import path moved to `charm.land/glamour/v2`; API stays close to v1 with a major-version bump for the module path[^5]. |

## References

[^1]: Glamour README — usage, color downsampling via Lip Gloss, and list of adopting projects (Glow, GitHub CLI, GitLab CLI, Gitea CLI). https://github.com/charmbracelet/glamour
[^2]: GitHub API repository metadata for charmbracelet/glamour — `created_at` 2019-12-18, MIT license, Go. https://github.com/charmbracelet/glamour
[^3]: goldmark — CommonMark-compliant Markdown parser for Go, used by Glamour to build the AST. https://github.com/yuin/goldmark
[^4]: Chroma — Go syntax highlighter used by Glamour for fenced code blocks. https://github.com/alecthomas/chroma
[^5]: Glamour v2 module import path `charm.land/glamour/v2`, per the README usage examples. https://github.com/charmbracelet/glamour
[lipgloss]: https://github.com/charmbracelet/lipgloss

## Tags

go, golang, markdown, cli, tui, terminal, ansi, rendering, charm, stylesheet, syntax-highlighting
