# tmux/tmux

> A terminal multiplexer: many terminals in one screen, detachable from and reattachable to a running session.

[GitHub repo](https://github.com/tmux/tmux) ·
[Project wiki](https://github.com/tmux/tmux/wiki) ·
[License: ISC](https://github.com/tmux/tmux/blob/master/COPYING)

## Overview

tmux lets a single terminal window host multiple sessions, windows, and panes,
and — crucially — lets those sessions keep running after the terminal that
started them is closed. Detach from a shell, close your laptop, SSH back in
tomorrow, reattach, and the processes are still there. This detach/reattach
property is why tmux is a near-universal fixture on remote servers: it survives
dropped SSH connections that would otherwise kill every foreground job.

It was written by Nicholas Marriott as a BSD-licensed alternative to GNU
Screen, and has been part of the OpenBSD base system for most of its life[^1].
That heritage shows: the codebase is portable C with two hard dependencies
(libevent for the event loop, ncurses/terminfo for terminal handling), an ISC
license, and a conservative attitude toward breaking changes — except for a
handful of config-format shifts (see Production Notes) that have burned nearly
everyone's `.tmux.conf` at some point.

The defining tension in tmux is discoverability versus control. Everything is a
command in a small internal language, bound to keys behind a prefix
(`C-b` by default). This makes it enormously scriptable and configurable, but
means a new user faces a blank screen and a manpage rather than menus or
mouse-first UX — the gap that newer multiplexers like Zellij explicitly target.

## Getting Started

```bash
# Debian/Ubuntu
sudo apt install tmux
# macOS
brew install tmux
# From release tarball (needs libevent + ncurses headers)
./configure && make && sudo make install
```

```bash
tmux new -s work        # start a named session
# ... do work, split panes with C-b % and C-b " ...
# detach: C-b d  (session keeps running in the background)

tmux ls                 # list running sessions
tmux attach -t work     # reattach later, even from a new SSH login
```

A minimal `~/.tmux.conf` that fixes the two most common annoyances:

```tmux
set -g mouse on              # click to select panes, scroll, resize
set -sg escape-time 10       # stop the prefix/ESC key from lagging in vim
set -g default-terminal "tmux-256color"
```

Reload without restarting: `tmux source-file ~/.tmux.conf`.

## Architecture / How It Works

tmux is a **client-server** program, which is the single most important fact for
understanding its behavior. Running `tmux` starts (or connects to) a long-lived
**server** process that owns all sessions and the actual PTYs of your programs.
The window you type in is a thin **client** attached to that server over a UNIX
domain socket (by default under `/tmp/tmux-<uid>/default`). Detaching just
disconnects the client; the server and your processes keep running. Killing the
server (`tmux kill-server`) destroys everything at once.

The object hierarchy is **session → window → pane**. A session is a set of
windows; a window is one "screen" divided into one or more panes; each pane is a
pseudo-terminal running a program (usually a shell). Windows and sessions are
independent of clients, so two people (or two terminals) can attach to the same
session and see the same thing, or attach to different windows of it.

Internally the server runs a libevent loop, multiplexing pane PTYs, client
sockets, and timers. Screen state for each pane is maintained in a grid model
that tmux re-renders to each attached client through terminfo, which is why tmux
can present a consistent `TERM` (e.g. `tmux-256color`) to programs regardless of
the outer terminal. Everything the user does maps to an internal **command**
(`split-window`, `select-pane`, `set-option`, …); key bindings are just commands
bound to keys, and the same commands are available from the shell (`tmux
split-window`) and from the `command-prompt`. **Control mode** (`tmux -CC`) exposes
this command/response stream as a machine-readable protocol, which is how
iTerm2's native tmux integration works.

## Production Notes

- **Config-format breaks are the classic footgun.** Two changes bit almost
  everyone: in 2.1 the many mouse options (`mode-mouse`, `mouse-select-pane`, …)
  collapsed into a single `mouse on`[^2]; in 2.9 the per-element style options
  (`window-status-bg`, `status-fg`, …) were replaced by unified `style` strings
  like `fg=colour1,bg=colour2`[^3]. A `.tmux.conf` copied from an old blog post
  will silently fail or error on a modern tmux. Distros ship widely varying
  versions, so a config that works on one server may not on another.
- **`escape-time` lag.** The default key-escape timeout makes the ESC key feel
  sluggish inside vim/neovim over tmux. Set `escape-time` to a small value
  (10ms) — a near-mandatory line for editor users.
- **TERM and true color.** Programs see `TERM=screen`/`tmux`; for 24-bit color
  you must both set `default-terminal` to a `*-256color` entry and add a
  `terminal-overrides`/`terminal-features` line advertising RGB (`Tc`) for the
  outer terminal. Getting colors "almost right" in tmux is a perennial support
  question.
- **Clipboard.** Copy-mode yanks into an internal paste buffer, not the system
  clipboard. Bridging to the OS clipboard needs OSC 52 support or an external
  tool (`pbcopy`, `xclip`, `wl-copy`) wired into `copy-pipe`.
- **Scrollback lives in the server.** Large `history-limit` values multiplied
  across many panes consume server memory that is not freed until panes close.
- **Server persistence has limits.** Sessions survive detach and SSH drops, but
  not a reboot or an OOM-killed server — there is no built-in session
  serialization. Restoring a layout after reboot needs a plugin
  (tmux-resurrect / tmux-continuum) or a startup script.
- **Nesting.** Running tmux inside tmux (common when SSHing into a box that also
  runs tmux) means the inner session swallows the prefix; the usual workaround is
  `C-b C-b` to send-prefix, or a distinct prefix for the inner server.

## When to Use / When Not

**Use when:**
- You run long jobs over SSH and need them to survive disconnects.
- You want scriptable, reproducible terminal layouts on servers.
- You share a live terminal session (pairing, teaching) via a shared session.
- You want one terminal to manage many shells without a GUI terminal's tabs.

**Avoid when:**
- You only ever work locally in a GUI terminal that already has tabs/splits and
  never need detach — the extra key-prefix layer is pure overhead.
- You want a mouse-first, discoverable UX out of the box — that is a non-goal.
- Your GUI terminal already has a built-in multiplexer with remote persistence
  (e.g. WezTerm), and you don't need tmux's ubiquity on the far end.

## Alternatives

- gnu/screen — the older multiplexer tmux was built to replace; use it when it's
  the only thing preinstalled or on ancient systems, accepting a crustier config
  language and slower development.
- zellij-org/zellij — Rust multiplexer with mouse support, on-screen keybinding
  hints, layouts, and WASM plugins; use it when discoverability and modern UX
  matter more than tmux's ubiquity.
- wez/wezterm — GPU terminal emulator with a built-in multiplexer and remote
  persistence; use it when you want terminal + multiplexer as one tool.
- dtach — minimal detach-only wrapper (no windows/panes); use it when all you
  need is to detach a single process, not a whole multiplexer.
- dvtm + abduco — a tiling window manager for the terminal plus a separate
  detach tool; use it when you want composable, single-purpose Unix tools.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2007-10 | First public release by Nicholas Marriott[^1]. |
| 1.0 | 2009-11 | First 1.x; adopted into OpenBSD base. |
| 2.1 | 2015-10 | Mouse options unified into a single `mouse` option[^2]. |
| 2.9 | 2019-04 | Style options unified into `fg=/bg=` strings[^3]. |
| 3.0 | 2019-09 | Requires a C99 compiler; format/expression changes. |
| 3.2 | 2021-05 | Popups and menus (`display-popup`), styled prompts. |
| 3.4 | 2024-02 | Maintenance + features release. |
| 3.5 | 2024-10 | Latest stable line (3.5a follow-up). |

## References

[^1]: tmux — history and origin, OpenBSD/BSD-licensed alternative to GNU Screen. https://github.com/tmux/tmux/wiki
[^2]: tmux CHANGES — 2.1 mouse-mode rework (`mode-mouse` etc. replaced by `mouse`). https://github.com/tmux/tmux/blob/master/CHANGES
[^3]: tmux CHANGES — 2.9 style-option rework (unified `style` strings). https://github.com/tmux/tmux/blob/master/CHANGES

## Tags

c, terminal, terminal-multiplexer, cli, ssh, developer-tools, session-management, tui, openbsd, unix
