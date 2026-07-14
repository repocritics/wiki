# charmbracelet/glow

> A terminal markdown reader that renders `.md` to styled ANSI, with a TUI file browser and a scriptable CLI.

[GitHub repo](https://github.com/charmbracelet/glow) ·
[License: MIT](https://github.com/charmbracelet/glow/raw/master/LICENSE)

## Overview

Glow is a command-line markdown viewer written in Go, first released in late 2019[^1] and part of Charm's TUI ecosystem (Bubble Tea, Glamour, Lip Gloss). It does one thing: take a markdown document — from a file, stdin, a URL, or a GitHub/GitLab repository reference — and render it to colored, word-wrapped ANSI text in your terminal. It ships in two modes: a CLI (`glow README.md`) for one-shot formatting and piping, and a TUI (`glow` with no args) that discovers markdown files in the current directory tree or Git repo and browses them in a `less`-style pager.

The important thing to understand is that Glow is a thin frontend. Nearly all of the actual rendering is done by [charmbracelet/glamour](https://github.com/charmbracelet/glamour), a separate library that parses markdown (via goldmark) and emits styled terminal output using JSON stylesheets. Glow contributes the file discovery, the pager, the TUI, and CLI flag handling; Glamour contributes the look. If you want to embed markdown rendering in your own Go program, you use Glamour directly — Glow is the end-user application built on top of it.

Its defining tradeoff is fidelity versus the terminal. Glow is not a browser: it approximates markdown within the constraints of a text grid. Headings, emphasis, code blocks with syntax highlighting, lists, blockquotes, and links all render well; tables, nested structures, and especially images do not translate cleanly to characters. Glow is excellent for reading prose docs and READMEs, and progressively less useful the more a document leans on visual layout.

## Getting Started

```bash
# macOS / Linux via Homebrew
brew install glow

# or install from source (requires Go 1.21+)
go install github.com/charmbracelet/glow/v2@latest
```

```bash
# Render a local file
glow README.md

# Pipe from stdin
echo '# Hello\n\n- rendered *inline*' | glow -

# Pull a README straight from a repo host
glow github.com/charmbracelet/glow

# Force a style and wrap width when auto-detection is wrong
glow -s dark -w 100 CHANGELOG.md

# Launch the TUI file browser
glow
```

## Architecture / How It Works

Glow sits at the top of the Charm stack, and understanding that stack explains most of its behavior:

- **goldmark** parses markdown into an AST (inside Glamour).
- **Glamour** walks the AST and renders each node to ANSI according to a style (a JSON stylesheet defining colors, margins, and prefixes). Code blocks are syntax-highlighted via chroma. The `-s` stylesheet you pass to Glow is a Glamour style.
- **Bubble Tea** provides the Elm-architecture event loop for the TUI (model/update/view). The pager, file list, and key handling are Bubble Tea components.
- **Lip Gloss / Bubbles** supply the styling primitives and reusable widgets (viewport, list, textinput).

The CLI path is straightforward: read source, hand it to Glamour with a chosen style and width, print (optionally through a pager). Style selection defaults to `auto`, which queries the terminal's background color to pick `dark` or `light`. Width defaults to wrapping; `-w 0` disables it.

The TUI path adds file discovery. Glow walks the working directory recursively (or, inside a Git repo, scans the repo) collecting markdown files, presents them in a filterable list, and opens the selected one in a scrollable pager whose keybindings mirror `less`. This local-first browsing is what remains of a larger vision: v1 shipped a "stash" feature that synced markdown to a Charm-hosted cloud service for cross-device access. That cloud integration was removed in v2.0.0 (2024)[^2], and Glow is now a purely local reader with no account, sync, or network state beyond ad-hoc URL/repo fetching.

## Production Notes

Glow is a developer utility, not a service, so "production" mostly means scripting and CI. The real footguns:

- **Style auto-detection is fragile.** The `auto` style probes the terminal background color. Over SSH, inside some tmux/screen configurations, or in CI logs, that probe fails or returns wrong, giving you a light theme on a dark terminal (or unreadable gray-on-gray). In any non-interactive or remote context, pin `-s dark` or `-s light` explicitly. Do not rely on auto in scripts.
- **Not a CommonMark WYSIWYG.** Tables wider than the wrap width get squeezed or mangled; deeply nested lists lose structure; images render as their alt text or a link, never inline (glow has no terminal-image protocol support). If your docs are table- or image-heavy, output quality drops noticeably.
- **Piping and TTYs.** When stdout is not a TTY, behavior differs from interactive use; ANSI codes may or may not be emitted depending on flags and detection. For deterministic output in a pipeline, set width and style explicitly rather than trusting terminal detection.
- **Remote fetch is a network dependency.** `glow github.com/owner/repo` and `glow https://…/file.md` reach out over the network and will hang or fail on restricted hosts. This is convenient interactively and a liability in automation.
- **No watch/live-reload.** Glow renders once. For a preview-on-save workflow you need an external file watcher (`entr`, `watchexec`) re-invoking it.
- **v1 → v2 is a breaking line.** The Go module path became `/v2`, and anyone who depended on the Charm cloud stash lost it. `go install …/glow@latest` (without `/v2`) pins you to the old major; use the `/v2` path for current releases.

## When to Use / When Not

**Use when:**
- You want to read a README, changelog, or docs file in the terminal without opening an editor or browser.
- You need styled markdown in a shell pipeline (`some-tool --help-md | glow -`).
- You want a quick TUI to browse the markdown scattered through a repo.
- You already use Charm tools and want a consistent look.

**Avoid when:**
- You need faithful rendering of tables, diagrams, or inline images — reach for a browser-based preview or an image-capable renderer.
- You're embedding markdown rendering inside your own Go program — use charmbracelet/glamour directly instead of shelling out to Glow.
- You need live preview on save — Glow renders once; pair it with a watcher or use an editor plugin.
- You want a general syntax-highlighting pager for arbitrary files — that's `bat`, not Glow.

## Alternatives

- charmbracelet/glamour — the rendering library underneath Glow; use it when you want to embed markdown-to-ANSI in your own Go application rather than run a CLI.
- swsnr/mdcat — Rust markdown terminal renderer; use it when you need inline images via iTerm2/kitty graphics protocols.
- sharkdp/bat — syntax-highlighting pager; use it when you want raw markdown source colored (not rendered to prose) alongside every other file type.
- Textualize/frogmouth — Python/Textual markdown browser TUI; use it when you want a mouse-driven, tabbed, table-of-contents reading experience.
- pandoc — universal document converter; use it when you need to transform markdown into other formats rather than view it.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1 | 2019-12-21 | First tagged release[^1]. |
| v1.0.0 | 2020-10-06 | First stable line; TUI + CLI. |
| v1.4.0 | 2021-03-14 | Stashing markdown to the Charm cloud service. |
| v1.5.0 | 2023-01-23 | Last v1 feature release. |
| v2.0.0 | 2024-08-23 | Module path `/v2`; Charm cloud stash removed; local-first reader[^2]. |
| v2.1.0 | 2025-02-26 | Feature release on the v2 line. |
| v2.1.2 | 2026-04-09 | Latest release as of this writing. |

## References

[^1]: charmbracelet/glow release history — first tag v0.1, 2019-12-21. https://github.com/charmbracelet/glow/releases
[^2]: charmbracelet/glow v2.0.0 release, 2024-08-23 (major version, `/v2` module path, cloud stash removed). https://github.com/charmbracelet/glow/releases/tag/v2.0.0

## Tags

go, cli, tui, markdown, terminal, markdown-renderer, charm, bubbletea, developer-tools, pager
