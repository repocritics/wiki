# crossterm-rs/crossterm

> Pure-Rust, cross-platform terminal manipulation library — the same code drives ANSI terminals on UNIX and the Windows console API on Windows.

[GitHub repo](https://github.com/crossterm-rs/crossterm) ·
[docs.rs](https://docs.rs/crossterm/) ·
[crates.io](https://crates.io/crates/crossterm) ·
[License: MIT](https://github.com/crossterm-rs/crossterm/blob/master/LICENSE)

## Overview

Crossterm is a low-level terminal library written by Timon Post, first published in 2018[^1]. It sits one layer above raw escape sequences: you emit typed commands (`MoveTo`, `SetForegroundColor`, `Clear`, `EnterAlternateScreen`) and crossterm decides how to realize them on the current platform. On UNIX that means writing ANSI/VT escape sequences; on Windows it means either VT sequences (Windows 10+ with virtual-terminal processing enabled) or direct `winapi` console calls on older systems. The stated compatibility floor is Windows 7[^1].

Its reason for existing is Windows. Unix-only alternatives like termion are simpler because they only ever write escape codes; crossterm accepts a heavier internal design in exchange for one API that works on both. That is the defining tradeoff — you pay in dependency surface and abstraction for not writing platform `#[cfg]` branches yourself.

Crossterm is deliberately not a widget toolkit. It gives you cursor movement, styling, screen/raw-mode control, and an input-event stream, and stops there. The layout, rendering, and widget layer is expected to come from something built on top — most visibly Ratatui, which uses crossterm as its default backend[^2]. If you are looking for buttons and tables, you want a higher-level crate; crossterm is the plumbing underneath it.

## Getting Started

```toml
[dependencies]
crossterm = "0.29"
```

```rust
use std::io::{stdout, Write};
use crossterm::{
    execute, queue,
    style::{Color, Print, ResetColor, SetForegroundColor},
    cursor::MoveTo,
    terminal::{Clear, ClearType},
};

fn main() -> std::io::Result<()> {
    let mut out = stdout();

    // execute! flushes immediately.
    execute!(out, Clear(ClearType::All), MoveTo(0, 0))?;

    // queue! buffers; you flush once. Fewer syscalls for bulk output.
    queue!(
        out,
        SetForegroundColor(Color::Blue),
        Print("styled line"),
        ResetColor
    )?;
    out.flush()?;
    Ok(())
}
```

Reading input requires raw mode, and raw mode must be turned back off before exit or the shell is left broken:

```rust
use crossterm::{
    event::{read, Event, KeyCode},
    terminal::{enable_raw_mode, disable_raw_mode},
};

fn read_keys() -> std::io::Result<()> {
    enable_raw_mode()?;
    loop {
        if let Event::Key(k) = read()? {
            if k.code == KeyCode::Char('q') { break; }
        }
    }
    disable_raw_mode()  // MUST run, including on panic paths
}
```

## Architecture / How It Works

The core abstraction is the `Command` trait. Every action is a zero-cost struct implementing `Command`, which knows how to write its own ANSI form and (on Windows) how to fall back to a `winapi` call. The `execute!` and `queue!` macros feed commands to any `io::Write`. `execute!` writes and flushes; `queue!` only buffers, letting you batch a full frame and flush once — the pattern Ratatui relies on to avoid a syscall per cell. The equivalent method-based API (`ExecutableCommand` / `QueueableCommand`) exists for non-macro call sites.

Platform dispatch is internal. On Windows the library attempts to enable virtual-terminal processing; if that succeeds it uses the same ANSI path as UNIX, and only drops to console-API calls where ANSI cannot express the operation. This is why RGB and 256-color output are gated to Windows 10+ and all UNIX terminals but not older Windows consoles — the capability, not the API, is the limit.

The event system is a separate subsystem behind the `events` feature (on by default). On UNIX it polls file descriptors through `mio` and handles terminal-resize `SIGWINCH` via `signal-hook`; the `poll`/`read` API lets you check for input with a timeout before blocking. The optional `event-stream` feature exposes a `futures::Stream<Item = Result<Event>>` for async runtimes. If you only need output and not input, disabling `events` (or using the `filedescriptor` feature) removes the `mio` / `signal-hook` dependencies and makes crossterm a very thin layer.

Historically crossterm was not one crate. It shipped as a family — `crossterm_style`, `crossterm_cursor`, `crossterm_input`, `crossterm_terminal`, `crossterm_screen`, plus `crossterm_winapi` and `crossterm_utils` — that were folded into a single unified crate during the early 0.x line[^3]. `crossterm_winapi` remains a separate published crate for the Windows console bindings.

## Production Notes

**Terminal state is global and must be restored.** Raw mode and the alternate screen are process-wide side effects on the real terminal. If your program panics or exits without calling `disable_raw_mode()` / `LeaveAlternateScreen`, the user's shell is left in a broken state (no echo, no line editing). Crossterm does not install a panic hook for you — the standard mitigation is a `std::panic::set_hook` (or a Drop guard) that restores the terminal before the default panic message prints.

**`queue!` requires an explicit flush.** A frequent bug is queuing output and never calling `flush()`, so nothing appears. `execute!` hides this by flushing every call, at the cost of a syscall per command — fine for one-off output, wasteful for full-screen redraws.

**Poll before you read.** `read()` blocks until an event arrives. Interactive loops that also need to do periodic work (redraw, tick timers) should use `poll(Duration)` to check for input without blocking, or the `event-stream` async API. Blocking naively on `read()` is the usual cause of a UI that won't repaint on resize or timer.

**Windows capability gaps are real.** Because the compatibility floor is Windows 7, code that assumes 256-color or RGB output will silently degrade on pre-Windows-10 consoles. Test on Windows Terminal and legacy conhost separately if you claim Windows support.

**Version churn.** Crossterm is pre-1.0 and every minor bump (0.27 → 0.28 → 0.29) can carry breaking API changes. The bigger operational hazard is transitive: because Ratatui and many TUIs pin a crossterm version, mixing a dependency built against one crossterm minor with your own use of another can produce two incompatible copies in the tree — align the version across the graph. Async users must also match the `event-stream` API to their runtime.

## When to Use / When Not

**Use when:**
- You need one terminal API that works on Windows and UNIX without hand-writing `#[cfg]` branches.
- You are building a TUI and want the low-level backend (or are using Ratatui, which defaults to it).
- You want fine control over buffering/flushing of terminal output.
- You need mouse events, resize events, or async input streams cross-platform.

**Avoid when:**
- You only ever target UNIX and want minimal dependencies — termion is leaner.
- You just need colored `println!`-style output — a styling-only crate is lighter.
- You want widgets, layout, or a full app framework — use a layer built on top, not crossterm directly.
- You need a stable, semver-frozen API — it is still pre-1.0 with breaking minor releases.

## Alternatives

- redox-os/termion — Unix-only, pure Rust, no external deps; use it when you never target Windows and want the smallest surface.
- ratatui/ratatui — higher-level TUI framework with widgets and layout; use it when you want to build screens, not emit raw commands (it sits on top of crossterm).
- wezterm/wezterm — its `termwiz` crate offers a richer terminal-capability model and its own line editor; use when you need advanced terminal features beyond crossterm's scope.
- console-rs/console — simple cross-platform styling, prompts, and progress UIs; use for CLI polish rather than full-screen apps.
- gyscos/cursive — a full TUI toolkit with its own backends; use when you want a batteries-included widget framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018 | Initial release; began as split `crossterm_*` sub-crates[^1][^3]. |
| 0.14 | 2019-12-16 | Consolidation era — unified single-crate API maturing[^3]. |
| 0.17 | 2020-03-24 | Reworked event/input model. |
| 0.20 | 2021-06-10 | Ongoing command-API and event refinements. |
| 0.25 | 2022-08-10 | Continued API stabilization. |
| 0.26 | 2023-01-28 | Feature and dependency cleanup[^4]. |
| 0.27 | 2023-08-06 | Version pinned in README examples[^1]. |
| 0.28 | 2024-07-31 | Minor release with breaking changes[^4]. |
| 0.29 | 2025-04-05 | Latest release as of writing[^4]. |

## References

[^1]: crossterm README, crossterm-rs/crossterm (master). https://github.com/crossterm-rs/crossterm
[^2]: Ratatui backend documentation — crossterm is the default backend. https://ratatui.rs/
[^3]: crossterm README, License section listing the historical sub-crates (`crossterm_screen`, `crossterm_cursor`, `crossterm_style`, `crossterm_input`, `crossterm_terminal`, `crossterm_winapi`, `crossterm_utils`). https://github.com/crossterm-rs/crossterm
[^4]: crossterm GitHub releases (tag dates via GitHub API). https://github.com/crossterm-rs/crossterm/releases

## Tags

rust, terminal, tui, cross-platform, cli, ansi, console, input-events, styling, library
