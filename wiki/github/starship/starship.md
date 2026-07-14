# starship/starship

> A cross-shell prompt written in Rust — one config, one binary, the same prompt on any shell.

[GitHub repo](https://github.com/starship/starship) ·
[Official website](https://starship.rs) ·
[License: ISC](https://github.com/starship/starship/blob/main/LICENSE)

## Overview

Starship is a prompt — the text your shell prints before each command — implemented as a single Rust binary that any shell can call[^1]. Instead of writing prompt logic in shell script (as oh-my-zsh themes or a hand-rolled `PS1` do), you install one executable and add one line to your shell's rc file. The binary inspects the current directory and environment and prints a formatted prompt string. The same binary and the same `~/.config/starship.toml` drive Bash, Zsh, Fish, PowerShell, Nushell, Elvish, Ion, Tcsh, Xonsh, and Cmd, which is the project's central value proposition: you configure your prompt once and it follows you across shells and machines.

The prompt is composed of **modules** — directory, git branch, git status, language version detectors (Node, Python, Rust, Go, etc.), package version, command duration, exit status, and dozens more. Each module renders only when relevant: the `nodejs` module appears when a `package.json` or `.js` file is present, the `git_branch` module only inside a repository. This "show relevant information at a glance" behavior is the default; nearly every aspect is overridable in TOML.

The defining tension is **per-prompt subprocess cost versus convenience**. Because Starship is an external binary invoked on every prompt render, it pays process-spawn overhead that an in-shell prompt does not, and slow modules (chiefly `git_status` in large repositories) can add visible latency. The project mitigates this with parallel module evaluation and configurable timeouts, but the architecture means prompt speed is something you occasionally tune rather than something that is always free. The README's "blazing-fast" framing is marketing; in practice Starship is fast enough for most users and tunable for the rest.

## Getting Started

Install the binary (see the README for per-OS package managers; `cargo install starship --locked` works anywhere Rust is available):

```sh
curl -sS https://starship.rs/install.sh | sh
```

Then hook it into your shell. For Zsh, add to the end of `~/.zshrc`:

```sh
eval "$(starship init zsh)"
```

For Bash use `~/.bashrc` with `eval "$(starship init bash)"`; for Fish add `starship init fish | source` to `~/.config/fish/config.fish`. A [Nerd Font](https://www.nerdfonts.com/) is required for the default glyphs to render — without one, icons appear as tofu boxes.

Configuration lives in `~/.config/starship.toml`:

```toml
# ~/.config/starship.toml
add_newline = false          # no blank line before the prompt
command_timeout = 1000       # ms budget for any single module's commands

[git_branch]
symbol = " "

[directory]
truncation_length = 3        # keep at most 3 path components

[python]
format = 'via [${symbol}${version}](yellow) '
```

## Architecture / How It Works

`starship init <shell>` prints a small shell snippet that wires Starship into that shell's prompt hook. The mechanism differs per shell — for POSIX shells it sets `PROMPT_COMMAND`/`precmd` to call `starship prompt`, passing context such as the previous command's exit status, the elapsed time, terminal width, and jobs count as flags. The heavy lifting happens in the Rust binary, not in shell code, which is why behavior is consistent across shells: only the thin init glue is shell-specific.

On each render, `starship prompt`:

1. Loads and merges configuration (built-in defaults overlaid with `starship.toml`).
2. Determines which modules are active for the current directory and environment.
3. Evaluates active modules **in parallel** — each module may stat files, read env vars, or shell out to tools like `git` or `python --version`.
4. Assembles the results according to each module's `format` string and the top-level `format`, then prints the final prompt.

Two configuration systems matter. **Format strings** (e.g. `format = 'via [$symbol$version]($style)'`) control layout and use `$variable`, `[text](style)`, and `(optional)` group syntax. **Styles** are a compact string grammar (`"bold green"`, `"#ff0000"`, `"bg:blue"`). Modules are independent and namespaced (`[git_status]`, `[nodejs]`, …); disabling one is `disabled = true`.

Git information is the most expensive area. Starship shells out to `git` and/or reads repository state to compute branch, ahead/behind counts, and the working-tree status (staged, modified, untracked, conflicts). In large or slow repositories this is the dominant cost of a render, which is why `git_status` has its own knobs and Starship exposes `scan_timeout` (filesystem scan budget) and `command_timeout` (external-command budget) to cap worst-case latency.

## Production Notes

**Per-prompt process spawn is the base cost.** Every prompt invokes the Starship binary anew; there is no long-lived daemon. On Linux and macOS this is a few milliseconds and rarely noticeable. On Windows, process creation is heavier, and users on slow disks or with many active modules are the ones most likely to perceive prompt lag.

**`git_status` in big repos is the usual culprit for slowness.** If your prompt feels sluggish, the first move is to profile with `starship timings`, which prints how long each module took. Common fixes: lower `command_timeout`, disable `git_status` (keeping the cheaper `git_branch`), or set `git_status.disabled = true` in monorepos. Network filesystems and repositories with huge working trees amplify the effect.

**Nerd Font dependency is a real onboarding footgun.** The default preset uses glyphs from Nerd Fonts; a fresh install in a terminal without one shows boxes or blank squares. There is an ASCII-friendly path (the "Plain Text Symbols" preset and per-module `symbol` overrides), but the out-of-box experience assumes a patched font installed *and selected in the terminal emulator* — two separate steps people miss.

**Config is not versioned or namespaced by shell.** One `starship.toml` applies everywhere. That is the point, but it means a module tuned for one environment (e.g. assuming a tool is on `PATH`) behaves identically in shells where that assumption fails. Use module `detect_*` conditions and `when`/`disabled` rather than expecting per-shell config.

**Upgrades are low-drama but watch config keys.** Starship follows semver post-1.0; minor releases add modules and occasionally deprecate config keys with warnings. Breaking config changes are rare but do happen across minor versions — read release notes if you pin a version in dotfiles. The install script always fetches the latest release, so reproducible setups should pin via a package manager or `cargo install --version`.

**Transient prompt and right prompt** exist but are shell-dependent. Features like transient prompt (collapsing past prompts to a minimal form) and `right_format` are supported on a subset of shells; check the docs before relying on them cross-shell.

## When to Use / When Not

**Use when:**
- You work across multiple shells or machines and want one prompt config everywhere.
- You want language/tool version indicators and git state without hand-writing prompt scripts.
- You value a declarative TOML config over shell-script prompt hacking.
- You're on Fish, Nushell, Elvish, or another shell whose native prompt theming is thin.

**Avoid when:**
- You live entirely in Zsh and want the absolute fastest prompt — powerlevel10k's instant-prompt and in-shell rendering will beat an external binary.
- You cannot install a Nerd Font (locked-down terminals) and don't want to maintain an ASCII override.
- Your environment forbids extra binaries or per-prompt subprocess spawns (some hardened/embedded shells).
- You need a prompt that also brings plugins, aliases, and completions — Starship is *only* a prompt, not a shell framework.

## Alternatives

- JanDeDobbeleer/oh-my-posh — the closest direct competitor: a cross-shell prompt engine (Go). Use it instead when you prefer its themeing model or Windows-first tooling.
- romkatv/powerlevel10k — use instead when you are Zsh-only and want the fastest possible prompt via instant-prompt and in-shell rendering.
- ohmyzsh/ohmyzsh — use instead when you want a full Zsh framework (plugins, aliases, completions), not just a prompt; its themes are Zsh-only.
- IlanCosman/tide — use instead when you are Fish-only and want a native, fast Fish prompt.
- denysdovhan/spaceship-prompt — use instead if you want the Zsh prompt that inspired Starship and are staying in Zsh.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019-08 | First public release; cross-shell prompt in Rust[^1]. |
| 1.0.0 | 2021-07 | First stable release; semver commitment[^2]. |
| 1.x | 2021–2026 | Steady module additions (new languages, VCS, cloud/context modules), Nushell/Cmd support, `starship timings`/profiling, ongoing performance work[^3]. |

## References

[^1]: Starship — cross-shell prompt project site and documentation. https://starship.rs
[^2]: Starship v1.0.0 release. https://github.com/starship/starship/releases/tag/v1.0.0
[^3]: Starship configuration reference (modules, format strings, timeouts). https://starship.rs/config/

## Tags

rust, shell-prompt, cli, cross-shell, developer-tools, zsh, bash, fish, powershell, terminal, dotfiles
