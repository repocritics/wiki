# zellij-org/zellij

> A terminal multiplexer written in Rust — batteries-included by default, extensible through WebAssembly plugins.

[GitHub repo](https://github.com/zellij-org/zellij) ·
[Official website](https://zellij.dev) ·
[License: MIT](https://github.com/zellij-org/zellij/blob/main/LICENSE.md)

## Overview

Zellij is a terminal workspace — a multiplexer in the lineage of tmux and GNU
screen — first released in 2021 after starting life under the name "Mosaic"[^1].
It splits a single terminal into panes and tabs, keeps sessions alive when the
client detaches, and lets multiple clients attach to the same session for real
multiplayer editing. Where tmux ships minimal and expects heavy `.tmux.conf`
investment, Zellij's stated design goal is a good experience out of the box:
mouse support, a visible status/tab bar, discoverable modal keybindings, and a
UI-hint line are all on by default.

The two features that most distinguish it from tmux are the **WebAssembly plugin
system** — the status bar, tab bar, and session manager you see are themselves
plugins, and you can write your own in any language that compiles to WASM[^2] —
and **UX affordances** tmux lacks: floating panes, stacked panes, and a built-in
web client that serves the session over HTTP so a terminal emulator is optional.

The defining tradeoff is weight versus polish. Zellij's on-by-default UI, Rust
rendering pipeline, and Wasm plugin host cost more CPU, memory, and input latency
than tmux's spartan C core. On a laptop this is invisible; on a heavily-shared
server or over a high-latency SSH link the difference is measurable. Zellij is
actively developed with a large open-issue backlog — the maintainers note that
open issues do not necessarily indicate bugs, only reporter feedback[^3].

## Getting Started

```bash
# Prebuilt binary via most OS package managers, or from source:
cargo install --locked zellij

# Try it without installing anything permanent:
bash <(curl -L https://zellij.dev/launch)
```

Configuration and layouts use the **KDL** document language[^4]. A layout file
declares a reusable pane arrangement and can pre-run commands:

```kdl
// ~/.config/zellij/layouts/dev.kdl
layout {
    pane split_direction="vertical" {
        pane                       // editor
        pane split_direction="horizontal" {
            pane                   // shell
            pane command="cargo" { // runs on layout start
                args "watch" "-x" "test"
            }
        }
    }
}
```

```bash
zellij --layout dev        # start a session from the layout
zellij attach my-session   # reattach after detaching (Ctrl-o d)
```

## Architecture / How It Works

Zellij runs a **client–server** model within a single machine. Starting a session
spawns a background server process that owns the PTYs and multiplexing state;
`zellij` clients attach to it over a Unix socket. Detaching leaves the server (and
all running programs) alive; reattaching — or attaching a second client for
collaboration — reconnects to the same server. This is the same split tmux uses,
but Zellij's server is a multithreaded Rust process where a screen thread, PTY
thread, and plugin thread communicate over channels.

**Modal keybindings.** Unlike tmux's single prefix key, Zellij's default keymap is
modal: `Ctrl-p` enters pane mode, `Ctrl-t` tab mode, `Ctrl-n` resize, `Ctrl-s`
scroll/search, and so on, with the available keys shown in the status bar. This is
more discoverable for newcomers and more disruptive for users with existing muscle
memory or apps that already bind those chords — remapping to a tmux-style single
prefix is a common first configuration step.

**Plugins are WebAssembly.** Zellij embeds a Wasm runtime (Wasmtime) and runs
plugins as sandboxed WASI modules. The default UI components — the status bar, tab
bar, the `strider` file browser, the session manager — are all plugins loaded at
startup, not hardcoded chrome[^2]. Third-party plugins subscribe to events (key
presses, pane updates, timers) and render into panes through a defined host API.
The Rust `zellij-tile` crate is the reference SDK, but any WASM-targeting language
works. Plugins are compiled artifacts, so they are fetched/cached rather than
interpreted from source.

**Session serialization.** Zellij can persist session layout to disk and resurrect
it — reopening the same tab/pane structure (and optionally re-running the commands
that started each pane) after a reboot or crash[^5]. This is opt-in behavior that
tmux only approximates through third-party plugins like `tmux-resurrect`.

## Production Notes

**Resource overhead vs tmux.** The most-cited operator caveat: Zellij uses more
CPU and memory than tmux, and adds perceptible input latency in some setups,
because of its rendering pipeline and plugin host. It is fine for a personal
workstation but a poor fit for a shared bastion host with hundreds of sessions
where tmux's footprint is the reason it was chosen. Benchmark for your own
workload before standardizing on it server-side.

**KDL migration.** Configuration and layouts moved from YAML to KDL in the 0.32
release (late 2022)[^4]. Configs and layouts written for pre-0.32 Zellij do not
load on modern versions; old blog posts and dotfile repos using the YAML syntax
are a recurring source of confusion. Always check that examples target KDL.

**Keybinding collisions.** The default `Ctrl-*` mode keys shadow shell and editor
shortcuts — `Ctrl-s` (terminal flow control / incremental search), `Ctrl-n`,
`Ctrl-b`, `Ctrl-o`. Programs running inside Zellij may not receive these keys
until you remap Zellij's modes or use the "unbind" / locked mode. Plan the keymap
before rolling Zellij out to a team.

**Scrollback and copy.** Copy/paste integrates with the system clipboard (OSC 52
where supported), but behavior over SSH, inside tmux-inside-zellij nesting, and
across terminal emulators varies. Large scrollback buffers increase per-pane
memory; there is no unlimited-history guarantee comparable to tmux's tunables.

**Pre-release `main`.** The maintainers explicitly warn against installing from the
`main` branch: it is pre-release, and running it can corrupt the cache used by
released versions, forcing a manual cache clear before the stable build works
again. Pin to tagged releases in any reproducible or CI environment.

**Web client.** Recent versions ship a built-in web client that serves the session
over HTTP. Treat this as a network-exposed service — bind it carefully and front
it with authentication; do not expose a terminal session to an untrusted network.

## When to Use / When Not

**Use when:**
- You want a multiplexer that is pleasant before you write any config.
- You value floating/stacked panes, session resurrection, or real-time collaboration.
- You want to script the UI in a real language via WASM plugins.
- Your work is on a personal machine or low-contention host where the overhead is irrelevant.

**Avoid when:**
- You run hundreds of sessions on a shared server and every MB / ms of latency counts — tmux is lighter.
- You have deep tmux muscle memory and config and see no reason to relearn a modal keymap.
- You need the broadest possible availability on locked-down/minimal servers where tmux or screen is already present.
- You require a decade-stable config format — Zellij still iterates on config (the YAML→KDL break is precedent).

## Alternatives

- tmux/tmux — the incumbent C multiplexer; far lighter and ubiquitous on servers, but minimal out of the box and configured through `.tmux.conf`.
- wez/wezterm — a terminal emulator with built-in multiplexing and Lua config; use it when you want the emulator and multiplexer to be one program.
- kovidgoyal/kitty — GPU terminal whose "kittens" and window/tab layout cover much multiplexing locally; use it when you don't need detached server-side sessions.
- gnu/screen — the original; use it only when it's the one tool already installed on a restricted host.
- Zombie/zellij-style TUIs aside, `dvtm` + `abduco` — minimalist split; use when you want the smallest possible footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-04 | First release under the name Zellij (renamed from Mosaic)[^1]. |
| 0.32.0 | 2022-12 | Configuration and layouts switch from YAML to KDL[^4]. |
| 0.38.x | 2023 | Session serialization / resurrection matures[^5]. |
| 0.40.x | 2024 | Stacked panes, pane/tab UX refinements. |
| 0.43.x | 2026 | Built-in web client; ongoing plugin API work. |

*Version dates are approximate where the changelog is the only source; see the
releases page for exact tags.*

## References

[^1]: Zellij — "A terminal workspace with batteries included." https://zellij.dev
[^2]: Zellij plugin system documentation (WebAssembly / WASI). https://zellij.dev/documentation/plugins.html
[^3]: Zellij README, "About issues in this repository." https://github.com/zellij-org/zellij
[^4]: Zellij configuration & layouts (KDL). https://zellij.dev/documentation/configuration.html
[^5]: Zellij session resurrection documentation. https://zellij.dev/documentation/session-resurrection.html

## Tags

rust, terminal, terminal-multiplexer, tmux-alternative, tui, cli, webassembly, wasm-plugins, developer-tools, workspace, kdl, session-management
