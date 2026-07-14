# ratatui/ratatui

> Immediate-mode terminal UI library for Rust — draw the whole screen every frame, let a buffer diff figure out the minimal writes.

[GitHub repo](https://github.com/ratatui/ratatui) ·
[Official website](https://ratatui.rs) ·
[License: MIT](https://github.com/ratatui/ratatui/blob/main/LICENSE)

## Overview

Ratatui is a Rust crate for building text user interfaces (TUIs): dashboards, interactive CLIs, and full-screen terminal apps. It is a community-maintained fork of `tui-rs`, created in 2023 after the original by Florian Dehau went unmaintained[^1]. As of 2026 it is the default choice for TUIs in Rust and the rendering layer behind a large share of the ecosystem's terminal apps (gitui, bottom, atuin, and many others).

The defining design decision is **immediate-mode rendering**. There is no retained widget tree, no component instances that persist across frames, and no observer/dirty-tracking system. On every frame you build ephemeral widget structs and render them into a `Buffer`; Ratatui diffs that buffer against the previously displayed one and emits only the cells that changed. This makes the mental model simple — the UI is a pure function of your application state — but it pushes real responsibilities onto you: you own the event loop, you decide when to redraw, and any widget state that must survive between frames (scroll offset, list selection, table cursor) lives in your own `State` structs, not in the widgets.

The second thing to understand is that Ratatui is deliberately *not* a batteries-included framework. It does not read input, manage an async runtime, or provide an application architecture. It draws widgets into a rectangle and diffs buffers; everything else — input via `crossterm`, error handling via `color_eyre`, async via `tokio` — is assembled by the application author. That minimalism is why it composes well and why first-time users often underestimate how much scaffolding a real app needs.

## Getting Started

```bash
cargo add ratatui crossterm
```

```rust
use std::io;
use crossterm::event::{self, Event, KeyCode};
use ratatui::{
    widgets::{Block, Paragraph},
    DefaultTerminal, Frame,
};

fn main() -> io::Result<()> {
    let mut terminal = ratatui::init();   // raw mode + alternate screen
    let result = run(&mut terminal);
    ratatui::restore();                   // ALWAYS restore, even on error
    result
}

fn run(terminal: &mut DefaultTerminal) -> io::Result<()> {
    loop {
        terminal.draw(draw)?;
        if let Event::Key(key) = event::read()? {
            if key.code == KeyCode::Char('q') {
                return Ok(());
            }
        }
    }
}

fn draw(frame: &mut Frame) {
    let widget = Paragraph::new("Hello. Press q to quit.")
        .block(Block::bordered().title("ratatui"));
    frame.render_widget(widget, frame.area());
}
```

`ratatui::init()` / `restore()` (added to reduce a long-standing footgun) wrap enabling raw mode and entering the alternate screen. The project also ships starter [templates](https://github.com/ratatui/templates) via `cargo generate ratatui/templates`.

## Architecture / How It Works

The core pieces are small and layered:

- **`Terminal<B: Backend>`** — owns two `Buffer`s (front and back). `terminal.draw(closure)` hands you a `Frame`, clears the back buffer, runs your render closure, then diffs back-vs-front and writes only changed cells to the backend.
- **`Backend`** — the abstraction over the actual terminal. `CrosstermBackend` is the default and only cross-platform option (Windows included); `TermionBackend` (Unix-only) and `TermwizBackend` also exist. The backend does raw cursor moves, styling escape sequences, and cell writes.
- **`Buffer`** — a flat `Vec<Cell>` grid of styled characters (`Rect`-addressed). This is the single source of truth for a frame; widgets don't draw to the terminal, they mutate cells in the buffer.
- **`Widget` / `StatefulWidget`** — traits with a `render(self, area: Rect, buf: &mut Buffer)` method. `Widget` is consumed by value; `StatefulWidget` also takes `&mut State` so stateful widgets (`List`, `Table`, `Scrollbar`) can read/write persistent cursor state you own.
- **`Layout`** — splits a `Rect` into child rects from a list of `Constraint`s (`Length`, `Min`, `Max`, `Percentage`, `Ratio`, `Fill`). Solving is done by an internal constraint solver; results are memoized in a layout cache so repeated identical splits are cheap.

Since ~2024 the crate has been split into a modular workspace — `ratatui-core` (traits, buffer, layout, style), `ratatui-widgets` (the built-in widget set), and the `ratatui` facade that re-exports them[^2]. This lets widget authors depend on the stable core without pulling in the whole widget library, and is described in the repo's `ARCHITECTURE.md`.

Rendering is single-threaded and synchronous. There is no built-in event loop, timer, or async integration; the common pattern is a loop that blocks on `event::read()` (or `poll` with a timeout for tick-based animation), and, for async apps, a `tokio` task that forwards crossterm events into a channel.

## Production Notes

**You own the redraw cadence.** Because rendering is immediate-mode, a naive `loop { draw(); }` with no blocking will pin a CPU core at 100%. Gate draws on input events plus an optional tick (`event::poll(timeout)`); only redraw when state actually changed. The buffer diff minimizes *terminal writes*, not the cost of rebuilding widgets.

**Terminal restoration is a real hazard.** If your app panics while in raw mode / alternate screen, the user's terminal is left broken (no echo, no cursor, garbled). Install a panic hook that calls `ratatui::restore()` before printing the panic — `color_eyre` with Ratatui's panic-hook integration is the standard fix. This is the single most common "my terminal is messed up" support issue.

**Unicode width is a persistent footgun.** Column layout depends on `unicode-width`; CJK characters, emoji, and zero-width joiners can occupy a different number of terminal cells than their `char` count suggests. Wide glyphs that straddle a `Rect` boundary, or emoji whose width the terminal emulator disagrees about, cause visible misalignment. Test with real multibyte content, not just ASCII.

**Widgets are ephemeral by design.** A widget is consumed on `render` and rebuilt next frame, so you cannot hold a reference to "the list widget" and mutate it — all durable state goes in a `*State` struct you keep. Newcomers from retained-mode GUI toolkits routinely fight this.

**Pre-1.0, breaking changes are frequent.** Ratatui has not reached 1.0; minor releases regularly include API breaks. The project maintains a dedicated `BREAKING-CHANGES.md` and changelog, but upgrades across two or three minors usually require code edits (widget constructors, `Style`/`Stylize` changes, layout constraint signatures have all shifted over time). Pin the version and read the breaking-changes doc before bumping.

**Backend choice has consequences.** Crossterm is the safe default (only one that works on Windows). Termion is Unix-only. Mouse capture, bracketed paste, and keyboard-enhancement flags (distinguishing key release, modified keys) vary by backend and by terminal emulator, so features like "detect Shift+Enter" are not universally available.

## When to Use / When Not

**Use when:**
- You're building a terminal app in Rust and want layout + a widget set without writing escape sequences by hand.
- You want the UI to be a pure function of application state (immediate mode fits state-machine and Elm-style architectures well).
- You need cross-platform terminal support including Windows (via crossterm).
- You want a minimal, composable core rather than an opinionated framework.

**Avoid when:**
- You want a batteries-included framework with input handling, an event loop, and app architecture provided — Ratatui gives you none of these; expect to assemble the scaffolding.
- You're not in Rust: Bubble Tea (Go) or Textual (Python) are more ergonomic in their languages.
- You need a stable, rarely-breaking API today — it's still pre-1.0.
- Your "TUI" is really a line-based CLI; `clap` + `indicatif` + `dialoguer` are lighter than a full-screen renderer.

## Alternatives

- fdehau/tui-rs — the unmaintained predecessor Ratatui forked from; use only to understand legacy code, not for new projects.
- gyscos/cursive — retained-mode, callback/widget-tree model backed by ncurses/crossterm; use when you prefer event callbacks over redrawing every frame.
- charmbracelet/bubbletea — Go's Elm-architecture TUI framework; use when your app is in Go rather than Rust.
- Textualize/textual — Python async TUI with CSS-like styling; use for rich Python terminal apps.
- crossterm-rs/crossterm — the lower-level terminal-manipulation crate Ratatui sits on; use directly when you need cursor/style control but no widgets or layout.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016 | `tui-rs` created by Florian Dehau (Ratatui's ancestor)[^1]. |
| 0.20.0 | 2023-03 | First release under the Ratatui fork after tui-rs stalled[^1]. |
| 0.26.0 | 2024-02 | Widget ref rendering, `Stylize` ergonomics, layout improvements. |
| 0.29.0 | 2024 | Latest 0.x line; modular `ratatui-core` / `ratatui-widgets` workspace[^2]. |

(Version dates before 0.20 predate the fork and belong to `tui-rs`. Exact 0.2x release days: see the changelog[^3].)

## References

[^1]: Ratatui README, "Acknowledgements" — forked from `tui-rs` by Florian Dehau in 2023. https://github.com/ratatui/ratatui#acknowledgements
[^2]: Ratatui `ARCHITECTURE.md` — crate organization and modular workspace structure. https://github.com/ratatui/ratatui/blob/main/ARCHITECTURE.md
[^3]: Ratatui `CHANGELOG.md` (generated by git-cliff from Conventional Commits). https://github.com/ratatui/ratatui/blob/main/CHANGELOG.md

## Tags

rust, tui, terminal, cli, ratatui, immediate-mode, widgets, crossterm, terminal-ui, dashboard
