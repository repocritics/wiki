# asciinema/asciinema

> Terminal session recorder that captures text, not pixels — recordings are replayable, diffable, embeddable, and a fraction of the size of screen video.

[GitHub repo](https://github.com/asciinema/asciinema) ·
[Official website](https://asciinema.org) ·
[License: GPL-3.0](https://github.com/asciinema/asciinema/blob/develop/LICENSE)

## Overview

asciinema is a command-line tool that records a terminal session by running a
program inside a pseudo-terminal and capturing everything written to the
terminal, plus timing, into a lightweight text file in the `asciicast` format
(`.cast`)[^1]. Because it stores the byte stream and delays rather than encoded
frames, a several-minute recording is typically kilobytes rather than the
megabytes a `.mp4` screen capture would need. The same recordings can be
replayed in a terminal, embedded on a web page via the separate asciinema
player, or uploaded to an asciinema server such as asciinema.org for sharing.
The project began in 2011 (originally "ascii.io") by Marcin Kulik and is still
authored by him[^2].

The project is really three loosely coupled pieces maintained under one org: the
**CLI recorder** (this repo), the **player** (a JavaScript widget that plays
`.cast` files in the browser), and the **server** (the hosting/upload backend
behind asciinema.org, self-hostable). They communicate only through the
asciicast format and an HTTP upload/stream protocol, so you can adopt any subset
— record with the CLI and never touch asciinema.org, or serve your own player
with no server at all.

The defining tension of this repo specifically is the ongoing **language
rewrite**. The current generation (3.x) lives on the `develop` branch and is
written in Rust; the previous generation (2.x), written in Python, lives on the
`python` branch[^3]. Anyone reading old blog posts, packaging notes, or distro
packages needs to know which generation they are looking at — install method,
supported features (live streaming and asciicast v3 are 3.x additions), and even
the default branch name differ.

## Getting Started

Install via a package manager (`brew install asciinema`, `apt install
asciinema`, `pipx install asciinema` for the 2.x line) or build the current Rust
version from source:

```sh
cargo install --locked --git https://github.com/asciinema/asciinema
```

Record, replay, and stream:

```sh
# record a session to a file (exits when the inner shell exits)
asciinema rec demo.cast

# replay it in the terminal
asciinema play demo.cast

# live-stream via the built-in HTTP server (LAN/localhost viewers)
asciinema stream -l

# live-stream through a relay (an asciinema server)
asciinema stream -r
```

Building the 3.x CLI requires the Rust compiler 1.82 or later and Cargo[^1].
asciinema runs on GNU/Linux, macOS, and FreeBSD; Windows is not supported[^4].

## Architecture / How It Works

The recorder allocates a **pseudo-terminal (PTY)**, runs your shell or command
as a child attached to it, and copies bytes in both directions while your real
terminal acts as a transparent pass-through. Output bytes plus timestamps are
written to the `.cast` file; input capture (keystrokes) is optional and
off by default.

The **asciicast format** is the actual contract of the ecosystem. v1 was a
single JSON document; v2 (shipped with the 2.0 line) switched to newline-
delimited JSON — a header object followed by one `[time, type, data]` event per
line — which made recordings streamable and appendable[^5]. The 3.x line
introduces asciicast v3 and can natively read and write zstd-compressed
recordings (`.zst`), which the README reports at roughly 8% of the original size
on average[^1]. The CLI can convert between v1/v2/v3, raw terminal output, and
plain text, and concatenate multiple recordings with timing adjusted
automatically.

Live streaming is the headline 3.x capability and comes in two modes: a
**built-in HTTP server** with an embedded web player (`stream -l`) for viewers
on the same LAN or localhost, and a **relay** mode (`stream -r`) that forwards
the stream to an asciinema server for remote viewers. A single session can
record to a file while streaming locally and remotely at the same time.

Everything the player renders is derived from the byte stream: it runs a
terminal emulator in the browser, so colors, cursor movement, and escape
sequences are reproduced rather than rasterized. This is why recordings are
selectable-text and re-themeable, and also why anything that depends on real GUI
output (image protocols, some TUI edge cases) may not reproduce faithfully.

## Production Notes

- **It records bytes, including secrets.** Anything printed to the terminal —
  tokens echoed by a script, `export`ed values shown by a prompt, paths, and
  hostnames — lands in the `.cast` file verbatim. Review recordings before
  publishing; there is no automatic redaction.
- **Optional input capture is a bigger footgun.** With keystroke capture on,
  passwords typed at prompts that disable echo can still be recorded from the
  input side. Leave it off unless you specifically need it.
- **Terminal environment matters.** The recording reflects `$TERM`, locale, and
  window size at record time. Playback in a smaller terminal than the recording
  can clip; 3.x adds configurable window size and optional auto-resize on
  playback to mitigate this.
- **Idle time.** Long pauses make replays tedious; `--idle-time-limit` (record
  or play) caps dead air. Set it deliberately — the default keeps real timing.
- **CI / scripted use.** Headless mode, configurable window size, and
  exit-status propagation make it usable in CI, but a program that detects it is
  not attached to an interactive TTY may behave differently under the PTY.
- **Generation mismatch.** Distro packages still frequently ship the Python 2.x
  CLI. If you need live streaming, asciicast v3, or zstd output, confirm you are
  on the Rust 3.x build; feature and flag differences between the two are the
  most common source of "that command doesn't exist" confusion.
- **Uploads default to a public host.** `asciinema upload` / the 3.x server
  integration targets asciinema.org unless configured otherwise; visibility
  controls exist, but self-host the server if recordings must not leave your
  infrastructure.

## When to Use / When Not

**Use when:**
- You want to demo CLI workflows, bug reproductions, or tutorials as
  copy-pasteable, re-themeable, lightweight recordings.
- You need to embed a terminal demo on a docs page or README without hosting a
  video file.
- You want live, low-bandwidth over-the-shoulder streaming of a terminal to
  reviewers or an audience.

**Avoid when:**
- You need to capture anything graphical — GUI apps, image-in-terminal
  protocols, or precise pixel output.
- The session contains secrets you cannot reliably scrub before sharing.
- Your audience needs audio/video narration synced to arbitrary on-screen
  visuals; a real screencast tool fits better (asciinema supports only an
  external synchronized audio URL, not embedded media).

## Alternatives

- charmbracelet/vhs — script terminal recordings from a `.tape` file and render
  to GIF/MP4/WebM; use when you want reproducible, code-defined output as video.
- faressoft/terminalizer — records and renders to animated GIF; use when a
  self-contained GIF (no player, no JS) is the deliverable.
- nbedos/termtosvg — records to a standalone animated SVG; use when you want an
  embeddable, dependency-free vector animation.
- icholy/ttygif (and `ttyrec`) — the classic `ttyrec` + GIF pipeline; use for
  minimal, old-school capture on constrained systems.
- Watfaq/PowerSession-rs — an asciinema-style recorder for Windows, which
  asciinema itself does not support[^4].

## History

| Version | Date | Notes |
|---------|------|-------|
| ascii.io | 2011 | Project started by Marcin Kulik; later renamed asciinema[^2]. |
| 1.x | ~2014 | Python CLI, asciicast v1 (single JSON document). |
| 2.0 | 2018 | asciicast v2 (newline-delimited JSON), streamable/appendable format[^5]. |
| 3.x | develop branch | Rust rewrite: live streaming, asciicast v3, zstd output, built-in HTTP player[^1][^3]. |

## References

[^1]: asciinema CLI README, features and build instructions. https://github.com/asciinema/asciinema
[^2]: asciinema about / history; project © 2011 Marcin Kulik. https://asciinema.org/about
[^3]: asciinema CLI README, "Development" — 3.x (Rust) on `develop`, 2.x (Python) on `python`. https://github.com/asciinema/asciinema
[^4]: asciinema CLI README, "Building" note — Windows unsupported; PowerSession suggested. https://github.com/asciinema/asciinema
[^5]: asciicast file format documentation. https://docs.asciinema.org/manual/asciicast/v3/

## Tags

rust, cli, terminal, screencast, recording, streaming, developer-tools, asciicast, gpl-3.0, terminal-recorder
