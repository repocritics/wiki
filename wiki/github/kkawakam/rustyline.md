# kkawakam/rustyline

> Readline implementation in pure Rust, modeled on Antirez's Linenoise — the default choice for line editing in Rust REPLs and interactive CLIs.

[GitHub repo](https://github.com/kkawakam/rustyline) ·
[crates.io](https://crates.io/crates/rustyline) ·
[License: MIT](https://github.com/kkawakam/rustyline/blob/master/LICENSE)

## Overview

Rustyline is a GNU-Readline-style line editor for terminal programs, started in 2015 as a Rust port of Antirez's minimalist Linenoise[^1]. Where Linenoise deliberately stops at ASCII and single-line editing, rustyline goes considerably further: full UTF-8 handling, word-level completion, filename completion, incremental history search (Ctrl-R), a kill ring, undo, hints, line wrapping for multi-line input, and both Emacs (default) and vi keymaps[^2]. If you type into a Rust-built REPL, there is a good chance rustyline is underneath it.

The project sits in an unusual spot: ~1.9k stars looks modest, but as a foundational dependency it punches far above that — it is the long-standing incumbent for the "read a line with editing" problem in Rust, with an ecosystem of derive macros and helper traits around it. Maintenance is active (last push July 2026, v18.0.1 released June 2026) but concentrated in a very small number of contributors, which shows up in the project's defining tradeoff: rapid, frequent major-version releases. Eight major versions shipped between early 2023 (v11) and early 2026 (v18)[^3], each with breaking API changes. Consumers get steady improvement; they also get regular migration work.

## Getting Started

```toml
[dependencies]
rustyline = "18"
```

```rust
use rustyline::error::ReadlineError;
use rustyline::{DefaultEditor, Result};

fn main() -> Result<()> {
    let mut rl = DefaultEditor::new()?;
    if rl.load_history("history.txt").is_err() {
        println!("No previous history.");
    }
    loop {
        match rl.readline(">> ") {
            Ok(line) => {
                rl.add_history_entry(line.as_str())?;
                println!("Line: {line}");
            }
            Err(ReadlineError::Interrupted) | Err(ReadlineError::Eof) => break,
            Err(err) => {
                println!("Error: {err:?}");
                break;
            }
        }
    }
    rl.save_history("history.txt")
}
```

`DefaultEditor` is the no-frills entry point. Custom behavior comes from implementing the `Helper` traits (below).

## Architecture / How It Works

Rustyline inherits Linenoise's core design decision: instead of a curses-style screen abstraction, it puts the terminal into raw mode and redraws the current input line(s) on each keypress. This keeps the dependency footprint small and plays well with programs that also print normal output, at the cost of owning all the escape-sequence bookkeeping itself.

- **Terminal backends.** Unix (termios via the `nix` crate; tested on Linux, macOS, FreeBSD) and the Windows console API (cmd.exe, PowerShell). The two backends have genuinely different capabilities — see Production Notes.
- **`Editor` + `Helper`.** The central type is `Editor<H: Helper, I: History>`. `Helper` is a supertrait bundling four extension traits: `Completer` (Tab completion), `Hinter` (grey inline suggestions), `Highlighter` (syntax coloring of the input line and prompt), and `Validator` (decides whether Enter submits or continues a multi-line input — this is how SQLite-style "keep reading until `;`" REPLs are built). The companion `rustyline-derive` crate provides derives so you only hand-implement the traits you care about.
- **History.** The `History` trait abstracts storage; a file-backed implementation is behind the `with-file-history` feature and an SQLite-backed one behind `with-sqlite-history`. History supports dedup, size limits, and readline-style incremental search.
- **Keymaps and bindings.** Emacs mode is the default; vi insert/command modes are complete enough to cover most muscle memory, including counts, `f`/`t` motions, and registers-lite yank behavior[^2]. The `custom-bindings` feature exposes `ConditionalEventHandler` for binding arbitrary key events to actions.
- **`ExternalPrinter`.** Lets other threads print above the active prompt without corrupting the line being edited — the standard answer for log-emitting interactive daemons.

The API is synchronous and blocking by design. There is no async variant; `readline()` parks the calling thread until Enter, Ctrl-C, or Ctrl-D.

## Production Notes

- **Major-version churn is the cost of admission.** v11 through v18 shipped in about three years[^3]. Breaking changes have included the fallible constructor (`Editor::new()` returning `Result`), the generic history parameter that produced `DefaultEditor`, and feature-flag reshuffles. Pin the major version in `Cargo.toml` and budget for periodic migrations; if several dependencies in your tree use rustyline, expect occasional version-skew friction.
- **Blocking API in async programs.** In a tokio/async-std application, run `readline()` inside `spawn_blocking` (or a dedicated thread) and feed lines through a channel. Trying to poll it on the runtime will stall the executor. If your whole UI is async-first, reedline or a custom crossterm loop may fit better.
- **Terminal state on panic/abort.** Raw mode is restored on normal return, but a panic that unwinds past the editor or an outright abort can leave the user's terminal in raw mode (no echo, no line buffering). Interactive tools that spawn subprocesses or can panic mid-read should install a panic hook or wrap execution to restore the TTY.
- **Windows is supported, with edges.** cmd.exe and PowerShell work; PowerShell ISE and Mintty (Cygwin/MinGW Git Bash) do not[^2]. Colors/highlighting require Windows 10+ unless forced under ConEmu. If your users live in Git Bash on Windows, test early — the failure mode is an unresponsive prompt, not an error.
- **Ctrl-C semantics change.** In raw mode, Ctrl-C surfaces as `ReadlineError::Interrupted` instead of delivering SIGINT's default process kill. Your loop must handle it explicitly, and any "Ctrl-C twice to quit" convention is yours to implement.
- **Feature flags matter.** File-backed history lives behind `with-file-history`; forgetting the flag (or building with `default-features = false`) makes `load_history`/`save_history` unavailable, which is why the README example wraps them in `cfg` guards.
- **120 open issues** against a small maintainer pool means niche terminal/emulator bugs (bracketed paste quirks, wide-character rendering in specific emulators) can sit for a while. The core editing path, by contrast, is well-exercised by heavy downstream users.

## When to Use / When Not

**Use when:**
- You are building a REPL, shell, or interactive CLI in Rust and want readline behavior (history, completion, Emacs/vi keys) without writing terminal code.
- You need multi-line input with validation (SQL-style continue-until-terminator).
- You want a synchronous, single-dependency-cluster solution that works on Unix and native Windows consoles.

**Avoid when:**
- Your application is async-first and the prompt is deeply integrated with an event loop — reedline or a hand-rolled crossterm loop avoids the `spawn_blocking` bridge.
- You need full-screen TUI behavior (panes, menus, mouse) — that is ratatui territory, not a line editor's.
- You are prompting for structured input (selects, confirms, passwords) rather than free-form lines — inquire/dialoguer are purpose-built for that.
- You cannot absorb breaking upgrades a couple of times per year and need a frozen API.

## Alternatives

- nushell/reedline — Nushell's line editor, built for that shell's needs; use it when you want a more batteries-included, actively evolving editor with menus and dynamic prompts.
- mikaelmello/inquire — use for structured prompts (select, multiselect, confirm, password) instead of free-form line editing.
- console-rs/dialoguer — similar structured-prompt niche, part of the console-rs family.
- antirez/linenoise — the C original; use when you are in C/C++ land and want ~1k lines with no dependencies.
- crossterm-rs/crossterm — the lower-level path; use when you need custom input handling and are willing to build editing yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015 | Initial release; Linenoise port begins[^1]. |
| 1.0.0 | 2016 | First stable tag. |
| 6.0.0 | 2020-01-05 | Mid-life cadence: roughly two majors per year from here on[^3]. |
| 10.0.0 | 2022-07-17 | Fallible `Editor::new()`; feature-flag reorganization era. |
| 11.0.0 | 2023-02-19 | Generic history backend; `DefaultEditor` alias; SQLite history option. |
| 13.0.0 | 2023-12-05 | Continued API iteration. |
| 15.0.0 | 2024-11-15 | Continued API iteration. |
| 17.0.0 | 2025-08-03 | Two patch releases followed within two months. |
| 18.0.0 | 2026-03-29 | Current major; v18.0.1 patch 2026-06-24[^3]. |

## References

[^1]: Rustyline README — "Readline implementation in Rust that is based on Antirez' Linenoise". https://github.com/kkawakam/rustyline#readme
[^2]: Rustyline README — supported platforms, feature list, and Emacs/vi keybinding tables. https://github.com/kkawakam/rustyline#readme
[^3]: Rustyline releases page (v2.1.0 2018 through v18.0.1 2026-06-24). https://github.com/kkawakam/rustyline/releases

## Tags

rust, readline, line-editing, repl, cli, terminal, tui-adjacent, history, completion, cross-platform
