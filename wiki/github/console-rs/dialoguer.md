# console-rs/dialoguer

> A Rust library for interactive command-line prompts — confirm, input, password, select, multi-select, sort, and fuzzy-select — built on the `console` terminal layer.

[GitHub repo](https://github.com/console-rs/dialoguer) ·
[Documentation](https://docs.rs/dialoguer) ·
[License: MIT](https://github.com/console-rs/dialoguer/blob/main/LICENSE)

## Overview

dialoguer is a small, focused crate for the "ask the user a question" slice of a CLI: yes/no confirmations, free-text input with validation and history, hidden password entry, single- and multi-selection menus, reorderable sort prompts, and fuzzy-filtered pickers. It is not a TUI framework — it draws line-oriented prompts inline in the terminal scrollback rather than taking over an alternate screen. That scope is the whole point: for a `cargo`-style tool that occasionally needs to prompt, dialoguer is a few lines of builder code, not an event loop.

The crate is part of the `console-rs` family originally created by Armin Ronacher (mitsuhiko) and now maintained under the `console-rs` GitHub org[^1]. It sits directly on top of `console` (terminal detection, styling, raw-mode key reads) and pairs naturally with `indicatif` (progress bars) from the same family. Because it delegates all terminal handling to `console`, dialoguer inherits that crate's cross-platform behavior on Windows, macOS, and Linux without carrying its own terminfo logic.

The defining tension is maturity versus versioning: dialoguer has been in use since 2017 and is stable in practice, yet it remains a `0.x` crate. Every minor bump (`0.10` → `0.11` → `0.12`) is allowed to break API under semver, and historically has. Downstreams pin exact minor versions and read the changelog before upgrading.

## Getting Started

```bash
cargo add dialoguer
# fuzzy-select and history live behind feature flags:
cargo add dialoguer --features fuzzy-select,history
```

```rust
use dialoguer::{theme::ColorfulTheme, Confirm, Input, Select};

fn main() -> std::io::Result<()> {
    let name: String = Input::with_theme(&ColorfulTheme::default())
        .with_prompt("Project name")
        .default("my-app".into())
        .interact_text()?;

    let items = ["Library", "Binary", "Workspace"];
    let kind = Select::with_theme(&ColorfulTheme::default())
        .with_prompt("Template")
        .items(&items)
        .default(0)
        .interact()?; // returns the selected index

    if Confirm::new()
        .with_prompt(format!("Create {} as a {}?", name, items[kind]))
        .interact()?
    {
        println!("scaffolding…");
    }
    Ok(())
}
```

Each prompt is a builder; `.interact()` (or `.interact_text()` for `Input`) blocks, reads keys, redraws the line, and returns the typed result — an index for `Select`, a `Vec<usize>` for `MultiSelect`, a parsed `T: FromStr` for `Input`.

## Architecture / How It Works

dialoguer is a thin, synchronous layer. There is no async, no runtime, no background thread: `.interact()` puts the terminal into raw mode via `console::Term`, loops on keypresses, and repaints the current prompt region using cursor-up + clear-line escape sequences. When the prompt resolves it restores the terminal and returns.

Rendering is routed through the `Theme` trait. `SimpleTheme` emits plain text; `ColorfulTheme` (the common choice) adds the checkmarks, colored prompts, and selection cursors most people associate with the crate. A prompt calls `theme.format_*` methods for each visual state (active item, selected item, error, final rendered answer), so custom look-and-feel means implementing `Theme` rather than patching the prompt types.

Notable internals and their consequences:

- **Prompt types are structs with builder methods**, consumed by `interact*`. `Input<T>` is generic over the parsed type and validates by re-prompting on `FromStr` failure or on a user `validate_with` closure.
- **`FuzzySelect`** filters items as you type using the `fuzzy-matcher` crate; it is gated behind the `fuzzy-select` feature to keep that dependency optional.
- **`Input` history** is behind the `history` feature via a `History` trait you implement (e.g. an in-memory ring buffer), enabling up-arrow recall.
- **`Editor`** shells out to `$EDITOR` for multi-line input rather than implementing an in-terminal editor.
- **Redraw is line-based, not full-screen.** Long select lists scroll within a paging window; the crate does not manage an alternate screen buffer, so prompt output stays in scrollback after completion.

Because every prompt owns the terminal only for the duration of `interact()`, dialoguer composes cleanly with ordinary `println!` output before and after — but it does not coordinate with a concurrently running `indicatif` progress bar unless you serialize them yourself.

## Production Notes

- **Non-interactive environments (CI, pipes) are a footgun.** With no TTY, raw-mode reads fail or return immediately; a prompt in a CI job can error or spin. Gate interactive paths on `console::Term::stdout().is_term()` (or an explicit `--yes`/`--non-interactive` flag) and supply defaults yourself.
- **`0.x` breaking changes are real.** The `0.10 → 0.11` line reworked interaction return types and theme methods; upgrades are not drop-in. Pin `=0.11.x` (or your chosen minor) and budget a code pass per bump.
- **Feature flags are load-bearing.** `FuzzySelect`, `Input` history, and completion are behind `fuzzy-select`/`history`/`completion` features. Code compiles fine until you reference a gated type; enable the feature in `Cargo.toml`, not just in `use`.
- **Ctrl-C / interrupt handling is your responsibility.** Depending on version and platform, an interrupt may return an `Err`, an empty value, or leave the terminal in raw mode. Handle the error and restore the terminal (or install a signal handler) rather than assuming a clean exit.
- **Windows terminals vary.** Behavior is generally good via `console`, but legacy `cmd.exe`, non-VT consoles, and some CI shells render escape sequences imperfectly. Test on the actual target terminal, not just Windows Terminal.
- **Not for dashboards.** Anything needing persistent multi-widget layout, live regions, or mouse input has outgrown dialoguer — reach for a full TUI crate before bending prompts into a UI.

## When to Use / When Not

**Use when:**
- You need a handful of interactive prompts (setup wizards, scaffolders, confirmations) inside an otherwise non-interactive CLI.
- You want prompts that leave clean scrollback output rather than taking over the screen.
- You are already in the `console`/`indicatif` ecosystem and want consistent styling.

**Avoid when:**
- You are building a full-screen, stateful terminal UI — use a real TUI framework.
- You need the richest possible prompt set (autocomplete, rich validation, date pickers) and prefer a `1.0`-stable API — evaluate inquire.
- Your program runs unattended; interactive prompts have no place in a non-TTY pipeline.

## Alternatives

- mikaelmello/inquire — use instead when you want a broader, more actively developed prompt set (autocomplete, richer validation, date/select variants) and a `1.0` API.
- console-rs/indicatif — use alongside, not instead: progress bars and spinners rather than prompts, from the same family.
- ratatui/ratatui — use instead when you need a full-screen, stateful TUI with layout and live widgets, not line prompts.
- clap-rs/clap — use instead when the input is really command-line arguments; reserve prompts for genuinely interactive gaps.
- console-rs/console — the terminal primitive layer under dialoguer; use directly when you only need styling and key reads, not prompt widgets.

## History

Note: GitHub's release dates for older tags were backfilled in a single batch and are not reliable; the table gives version lineage with dates only where independently known.

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | Repo created under mitsuhiko; core Confirm/Input/Password/Select prompts[^2]. |
| 0.10.0 | ~2022 | Theme and interaction API refinements; fuzzy-select maturing behind its feature. |
| 0.11.0 | ~2023 | Breaking interaction/return-type and theme changes; pre-1.0 semver churn. |
| 0.12.0 | recent | Latest published minor as of 2026-07; still `0.x`[^3]. |

## References

[^1]: dialoguer README and `console-rs` organization. https://github.com/console-rs/dialoguer
[^2]: Repository metadata, created 2017-05-09 (GitHub API `repos/console-rs/dialoguer`). https://github.com/console-rs/dialoguer
[^3]: Release tags via GitHub API; latest tag `v0.12.0`. https://github.com/console-rs/dialoguer/releases

## Tags

rust, cli, terminal, prompt, interactive, tui, user-input, console-rs, developer-tools
