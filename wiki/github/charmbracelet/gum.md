# charmbracelet/gum

> Composable interactive prompts and text styling for shell scripts — a Bubble Tea toolkit you drive from bash instead of Go.

[GitHub repo](https://github.com/charmbracelet/gum) ·
[Charm](https://charm.sh) ·
[License: MIT](https://github.com/charmbracelet/gum/blob/main/LICENSE)

## Overview

Gum is a single Go binary that exposes Charm's terminal-UI stack — Bubble Tea, Bubbles, and Lip Gloss — as a set of subcommands you call from a shell script[^1]. Each subcommand (`choose`, `filter`, `input`, `confirm`, `spin`, `style`, and about a dozen others) renders an interactive widget, then writes the user's selection to stdout and signals the outcome via exit code. The design goal is to let dotfile and glue-script authors get fuzzy finders, multi-select menus, spinners, and bordered layouts without writing or compiling any Go.

The defining tension is that gum is a UI layer bolted onto the Unix pipe model. It works because it keeps a strict contract: the widget draws on `/dev/tty` (or stderr), and only the chosen value goes to stdout, so `x=$(gum choose ...)` behaves like any other command substitution. That contract is also the source of most surprises — a script that pipes gum's output somewhere and also expects the TUI to appear has to understand which stream is which, and gum is only useful when a real terminal is attached, which makes it awkward inside CI, cron, and non-interactive pipelines.

It is aimed at people writing personal tooling: git helpers, install scripts, `fzf`-style pickers, deploy confirmations. It is not a TUI application framework — for a full-screen program with custom state you drop down to Bubble Tea directly.

## Getting Started

```bash
brew install gum          # macOS / Linux
go install github.com/charmbracelet/gum@latest   # from source
# also: pacman -S gum, dnf install gum, winget install charmbracelet.gum
```

```bash
#!/usr/bin/env bash
# A conventional-commit helper built entirely from gum subcommands.
TYPE=$(gum choose "fix" "feat" "docs" "refactor" "test" "chore")
SCOPE=$(gum input --placeholder "scope")
SCOPE=${SCOPE:+"($SCOPE)"}
SUMMARY=$(gum input --value "$TYPE$SCOPE: " --placeholder "Summary of change")
DESCRIPTION=$(gum write --placeholder "Details of this change (ctrl+d to finish)")

gum confirm "Commit changes?" && \
  git commit -m "$SUMMARY" -m "$DESCRIPTION"
```

Each line blocks until the user acts. `gum confirm` exits `0` on yes and `1` on no, so the `&&` short-circuits naturally.

## Architecture / How It Works

Gum is a thin dispatch layer. The CLI is parsed with Kong (struct-tag-driven flags), and each subcommand constructs a Bubble Tea model from the corresponding Bubbles component — `filter` wraps the list/fuzzy component, `input` and `write` wrap textinput/textarea, `spin` wraps the spinner, `table` wraps the table component — then runs the Bubble Tea event loop until the user commits or aborts. Styling is Lip Gloss all the way down, which is why every visual flag comes in `.foreground` / `.background` / `.border` families.

The stream discipline is the core mechanism. Bubble Tea renders to stderr (or the controlling tty) so that stdout stays clean for the one value the script actually wants. This is what makes `sel=$(ls | gum filter)` compose: stdin feeds the candidate list, the interactive UI paints on the terminal, and stdout carries only the picked line. Multi-select commands (`choose --no-limit`, `filter --no-limit`) emit one selected item per line.

Configuration has three layers with a fixed precedence: built-in defaults, then `$GUM_<COMMAND>_<FLAG>` environment variables, then explicit `--flags`, with later layers overriding earlier ones[^2]. This lets you set a house style once in your shell profile (`export GUM_INPUT_PROMPT_FOREGROUND="#0FF"`) and still override per-invocation. There is no config file format; the environment is the config surface.

Because every widget is a separate process invocation, gum has no shared session state. A script that shows five prompts spawns and tears down five Bubble Tea programs. That keeps each command trivially composable but means gum cannot express a single coordinated multi-panel UI — for that you write a real Bubble Tea program.

## Production Notes

- **Requires a TTY.** Gum needs a controlling terminal to render and read keys. In CI, cron, or `docker build`, the interactive commands will fail or hang. Guard with `[ -t 0 ]` and provide non-interactive fallbacks; some commands accept flags (`--timeout`, default selections) that make them degrade more gracefully, but there is no universal `--no-tty` mode.
- **Exit codes are the control flow.** `confirm` returns 1 on "no", and any command returns a non-zero code (typically 130) when the user hits `ctrl+c` or `esc`. Under `set -e` this aborts your whole script, which is usually correct for a cancel but bites people who wanted to treat cancel as a branch. Capture `$?` explicitly when cancel is a valid path.
- **stdout is data, not decoration.** Do not `echo` progress to stdout around a `x=$(gum ...)` capture — it will be concatenated into the captured value. Send your own chatter to stderr or use `gum log` / `gum style` which respect the split.
- **Version skew across package managers.** Distro packages (apt, dnf, some brew snapshots) frequently lag the GitHub releases by several minor versions, so flags shown in the current README may not exist in the binary you installed. Check `gum --version` and `gum <cmd> --help` against your installed build rather than the web docs.
- **Color depth depends on the terminal.** Lip Gloss auto-detects terminal capabilities (truecolor vs 256 vs no-color) and degrades. `NO_COLOR` and `CLICOLOR_FORCE` are honored. Output redirected to a non-terminal drops styling, which is intended but occasionally surprising when logging.
- **Performance.** For large candidate lists `gum filter` and `gum choose` are fine into the tens of thousands of lines, but they read all of stdin before rendering, so streaming an unbounded producer into them will block until it closes. `fzf` remains faster and more featureful for very large or streaming fuzzy-find workloads.

## When to Use / When Not

**Use when:**
- You want interactive menus, prompts, spinners, or styled output in a bash/zsh script without compiling anything.
- You're building dotfile aliases, git helpers, or install/deploy scripts for humans at a terminal.
- You want a consistent look (Lip Gloss borders, Charm palette) across many small scripts via environment variables.

**Avoid when:**
- The script runs unattended (CI, cron, containers, hooks) — there is no terminal for gum to draw on.
- You need one cohesive full-screen TUI with shared state — write a Bubble Tea program instead.
- You need the fastest possible fuzzy finder over huge or streaming input — `fzf` is the specialist tool.

## Alternatives

- junegunn/fzf — use instead when you only need best-in-class fuzzy selection over large or streaming lists and don't care about prompts, spinners, or styling.
- charmbracelet/bubbletea — use instead when you need a real stateful, multi-panel TUI application rather than one-shot script widgets.
- dialog / whiptail — use instead when you want the classic ncurses box-drawing dialogs and need to run on minimal legacy systems without a Go binary.
- charmbracelet/glow — use instead when the specific need is rendering Markdown files in the terminal (gum's `format` is a lighter subset).
- stone-payments/gum-like shell prompt libs / bash `select` — use the builtin `select` when a plain, unstyled menu suffices and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-06 | First public release; repo created 2022-06-10[^3]. |
| 0.x series | 2022–2024 | Command set grew to include `choose`, `filter`, `input`, `write`, `confirm`, `spin`, `style`, `join`, `format`, `pager`, `file`, `table`, `log`. |
| ongoing | 2026 | Actively maintained under Charm; ~24k stars, last push 2026-07-13[^3]. |

Gum has stayed pre-1.0 throughout its life; the project treats subcommand behavior as fairly stable while reserving the right to adjust flags between minor releases. Verify exact per-version changes against the GitHub releases page rather than this table.

## References

[^1]: Gum README — "A tool for glamorous shell scripts. Leverage the power of Bubbles and Lip Gloss in your scripts and aliases without writing any Go code." https://github.com/charmbracelet/gum
[^2]: Gum README, "Customization" — flags override values set with environment variables (`$GUM_<COMMAND>_<FLAG>`). https://github.com/charmbracelet/gum#customization
[^3]: GitHub API metadata for charmbracelet/gum (created_at 2022-06-10, pushed_at 2026-07-13, 24,027 stars, MIT, Go). https://github.com/charmbracelet/gum

## Tags

go, cli, tui, shell, bash, terminal, interactive, prompt, scripting, charm, bubbletea, dotfiles
