# sxyazi/yazi

> A terminal file manager written in Rust, built around non-blocking async I/O and a Lua plugin system.

[GitHub repo](https://github.com/sxyazi/yazi) ·
[Official website](https://yazi-rs.github.io) ·
[License: MIT](https://github.com/sxyazi/yazi/blob/main/LICENSE)

## Overview

Yazi is a TUI file manager in the lineage of ranger, lf, and nnn, first published in mid-2023[^1]. The name means "duck" in Chinese. It occupies the "modern Rust rewrite" slot in the terminal-tooling wave alongside tools like ripgrep, fd, and bat, and has grown quickly — ~40k GitHub stars as of this writing — because it ships features that older managers left to configuration or external scripts: async previews, image rendering, code highlighting, fuzzy integration, and a plugin runtime.

The defining design choice is that essentially every I/O and CPU operation is asynchronous and off the UI thread[^2]. Directory reads, file previews, image decoding, and search all run as scheduled tasks with progress reporting and cancellation, so the interface stays responsive on slow disks, huge directories, or network mounts where ranger visibly stalls. The cost of that choice is complexity: Yazi is a client-server architecture with a task scheduler, a pub/sub data layer, and a Lua VM embedded for customization — considerably more machinery than the single-binary minimalism of lf or nnn.

The honest tension: Yazi is explicitly **public beta** and its own README warns to "expect breaking changes"[^1]. It is stable enough that many people use it as a daily driver, but configuration formats, plugin APIs, and keybinding schemas have shifted between releases, and there is no 1.0 stability promise yet. You adopt it for the feature set, not for a frozen contract.

## Getting Started

```bash
# macOS
brew install yazi ffmpeg sevenzip jq poppler fd ripgrep fzf zoxide resvg imagemagick font-symbols-only-nerd-font

# Arch Linux
pacman -S yazi ffmpeg 7zip jq poppler fd ripgrep fzf zoxide imagemagick

# From source (needs a recent stable Rust toolchain)
cargo install --locked yazi-fm yazi-cli
```

Yazi ships two binaries: `yazi` (the file manager, crate `yazi-fm`) and `ya` (the CLI / package manager, crate `yazi-cli`). To make Yazi change your shell's directory on exit, wrap it — this is the intended pattern, since a child process cannot alter its parent's `cwd`:

```bash
# ~/.bashrc / ~/.zshrc
function yy() {
	local tmp="$(mktemp -t "yazi-cwd.XXXXXX")" cwd
	yazi "$@" --cwd-file="$tmp"
	if cwd="$(command cat -- "$tmp")" && [ -n "$cwd" ] && [ "$cwd" != "$PWD" ]; then
		builtin cd -- "$cwd"
	fi
	rm -f -- "$tmp"
}
```

## Architecture / How It Works

Yazi is not a monolith; it is a Cargo workspace of many crates (`yazi-fm`, `yazi-core`, `yazi-adapter`, `yazi-scheduler`, `yazi-plugin`, `yazi-widgets`, `yazi-cli`, and more). Rendering is done with [ratatui](https://github.com/ratatui/ratatui), the standard Rust TUI library.

Key subsystems:

- **Async scheduler.** All blocking work — reading directories, computing sizes, previews, bulk operations — is dispatched to a task pool with priorities, live progress, and cancellation[^2]. This is what keeps the UI thread free and is the direct source of the "fast" claim.
- **Preview / preload pipeline.** Files are pre-loaded before you land on them. Text is syntax-highlighted (via a bundled highlighter), images are decoded natively, and heavier types (video, PDF, archives) are handled through external tools (`ffmpeg`, `poppler`, `7z`). Missing those binaries silently degrades previews rather than erroring.
- **Image adapter.** Yazi detects the terminal and picks an image protocol: Kitty graphics, iTerm2/WezTerm inline images, or Sixel, with Überzug++ (X11/Wayland) and Chafa (ASCII fallback) as external backends[^1]. This is the most terminal-dependent part of the product and the most common source of "images don't show" reports.
- **Data Distribution Service (DDS).** A client-server model that needs no extra server process, with a Lua pub/sub layer for cross-instance communication and state persistence — how multiple Yazi instances stay coordinated and how plugins react to events.
- **Lua plugin system.** UI, previewers, preloaders, spotters, and fetchers are all scriptable in Lua. Themes and plugins install via `ya pkg` (the package manager), which can pin versions[^3]. The official plugin/flavor collection lives in a separate repo under the `yazi-rs` org.

The coupling story: the async scheduler, the Lua runtime, and the DDS pub/sub layer are interdependent, and the plugin API is defined against internal data structures. That is why plugin/config breakage tends to cluster around releases — the extensibility surface is wide and not yet frozen.

## Production Notes

- **Beta means beta.** Config (`yazi.toml`, `keymap.toml`, `theme.toml`) and the Lua plugin API have changed shape across releases. Pin the Yazi version and your plugins/flavors together (`ya pkg` supports version pinning) and read release notes before upgrading — a `keymap.toml` from an older version can fail to load.
- **External binaries are load-bearing.** Previews for images, video, PDF, and archives depend on `ffmpeg`, `imagemagick`/`resvg`, `poppler`, `7z`/`sevenzip`. Fuzzy features depend on `fd`, `ripgrep`, `fzf`, `zoxide`. None are vendored; a minimal install gives you a working manager with quietly missing previews. Install the full dependency set the docs list.
- **Image preview is terminal-specific.** Rendering only works if your terminal implements one of the supported protocols. Under tmux, multiplexers, or SSH the image path is the first thing to break; the Überzug++ and Chafa fallbacks exist precisely for terminals without native support. Budget time to get this right per-environment.
- **Nerd Font required for the default look.** The default theme uses glyphs from a Nerd Font (symbols-only is enough). Without one you get tofu boxes for icons.
- **Windows support exists but trails.** Yazi runs on Windows and images work in recent Windows Terminal via Sixel, but the platform is less battle-tested than Linux/macOS and some plugins assume Unix tools.
- **`cd on exit` needs the wrapper.** The shell function above (or the official `ya` shell integrations) is mandatory if you expect Yazi to leave you in the directory you navigated to — this trips up nearly every new user.

## When to Use / When Not

**Use when:**
- You want async, non-blocking navigation that stays responsive on large or slow directories.
- You want built-in image/video/PDF preview and code highlighting without hand-assembling a preview script.
- You want a scriptable (Lua) file manager and a real package manager for plugins and themes.
- You already live in the ripgrep/fd/fzf/zoxide ecosystem and want tight integration.

**Avoid when:**
- You need a frozen, 1.0-stable config that never breaks — Yazi is explicit beta.
- You want a single static binary with zero external dependencies (nnn or lf fit better).
- Your terminal can't do any supported image protocol and previews are the reason you're switching.
- You're on a locked-down or exotic platform where installing ffmpeg/poppler/imagemagick isn't practical.

## Alternatives

- gokcehan/lf — single-binary Go file manager, minimal, config-driven; use when you want lightweight and stable over feature-rich.
- ranger/ranger — the Python predecessor Yazi is often compared to; use when you want a mature, heavily-documented tool and don't mind synchronous, slower previews.
- jarun/nnn — extremely fast, tiny C manager with plugin scripts; use when you want the leanest possible footprint.
- Vifm/vifm — vi-like manager with a two-pane workflow; use when you want vi-modal editing semantics for files.
- kamiyaa/joshuto — another Rust ranger-like TUI manager; use when you want a Rust option that is simpler and less plugin-heavy than Yazi.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-07-08 | First public commit; repository created[^4]. |
| 0.1.x | 2023–2024 | Async core, image protocols, early plugin support. |
| 0.2.x | 2024 | Plugin system and package manager (`ya pkg`) maturation[^3]. |
| 0.3.x | 2024–2025 | DDS / client-server data layer, config and API refinements. |
| 0.4.x | 2025 | Continued feature work; still public beta, breaking changes ongoing[^1]. |

(Yazi has not reached 1.0; exact minor-version dates move quickly — check the releases page for specifics before relying on any single version.)

## References

[^1]: Yazi README and project status — "Public beta… expect breaking changes." https://github.com/sxyazi/yazi
[^2]: "Why is Yazi Fast?" — official explanation of the async architecture. https://yazi-rs.github.io/blog/why-is-yazi-fast
[^3]: Package manager documentation (`ya pkg`, plugins, flavors). https://yazi-rs.github.io/docs/cli
[^4]: GitHub repository metadata, `created_at` 2023-07-08. https://github.com/sxyazi/yazi

## Tags

rust, tui, file-manager, terminal, cli, async, developer-tools, lua-plugins, cross-platform, ratatui
