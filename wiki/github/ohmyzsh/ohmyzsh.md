# ohmyzsh/ohmyzsh

A community-driven framework for managing your Zsh shell configuration — 300+ plugins, 140+ themes, and an auto-updater that makes living on the bleeding edge low-friction.

## What it is

A shell configuration framework that turns Zsh from "a shell with potential" into "a shell with batteries included". Drops `~/.oh-my-zsh/` into the home directory and ships an opinionated `.zshrc` template that loads plugins and a theme. Plugins are largely zsh-script bundles for common tools (git, docker, kubectl, npm, etc.) and themes are prompt configurations. Has been the default starting point for new Zsh users since the early 2010s and ships preinstalled in some distros' developer tooling.

## Key features

- 300+ optional plugins covering git, docker, kubectl, npm, brew, rails, macOS, python, php, terraform, and most common dev tooling.
- 140+ themes ranging from minimal to information-dense (Powerlevel10k-compatible variants among them).
- Auto-update tool — `omz update` keeps the framework, plugins, and themes synced to upstream.
- 2,500+ contributors per the README — the largest community-contributor count among shell-config frameworks.
- MIT-licensed.

## Tech stack

- Shell only (Zsh + POSIX-friendly scripts).
- No build tooling; installation is a shell script.

## When to reach for it

- You're new to Zsh and want a sensible default configuration that exposes common conveniences without writing them yourself.
- You're standardizing developer environments and want a baseline that's broadly familiar.
- You want plugin auto-loading without writing your own dotfile-management plumbing.

## When *not* to reach for it

- You want the absolute fastest startup time — the framework adds overhead vs. a hand-tuned `.zshrc`.
- You want full control over your shell setup — graduating to a custom config (or to `zinit`/`zplug`) is the usual path for power users.
- You're not using Zsh — this is Zsh-specific and won't help bash/fish/nushell users.

## Maturity signal

187k stars, 26k forks, MIT, last push 2026-06-01 — actively maintained for over 15 years. The 2,500+ contributor count signals durable community velocity rather than single-author dependency. Open-issues count of 551 reflects plugin-by-plugin breakage requests across the long tail of integrations rather than core framework defects.

## Alternatives

- `zsh-users/zsh-syntax-highlighting` + `zsh-autosuggestions` + custom `.zshrc` — use when you want minimal overhead and full control.
- `romkatv/powerlevel10k` — use when you want only the prompt theme and nothing else.
- `zdharma-continuum/zinit` — use when you want fast, lazy plugin loading without the OMZ framework overhead.
- `starship/starship` — use when you want a cross-shell prompt (Zsh + Fish + Bash + PowerShell + Nu).

## Notes

The framework's age (2009 origin) and community size means it's the lingua franca for dotfile sharing — most tutorials assume you've installed OMZ. The startup-cost trade-off is real for long-running shells with many plugins; `omz` benchmark helpers exist. License (MIT) makes derivative theme/plugin distributions clean.

## Tags

shell, zsh, command-line-interface, plugin, theme, dotfiles, productivity, developer-tools, configuration
