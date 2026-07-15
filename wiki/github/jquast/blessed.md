# jquast/blessed

> A terminal-capability library for Python — colors, styles, cursor positioning, and keyboard input without the curses ceremony.

[GitHub repo](https://github.com/jquast/blessed) ·
[Documentation](https://blessed.readthedocs.io/en/latest/) ·
[PyPI](https://pypi.org/project/blessed/) ·
[License: MIT](https://github.com/jquast/blessed/blob/master/LICENSE)

## Overview

Blessed is a thin, high-level wrapper over the system `terminfo(5)` database that lets Python programs emit colors, text styles, and cursor movement, and read keyboard input, without directly touching the standard-library `curses` module[^1]. It is a maintained fork of Erik Rose's `blessings`[^2], carrying the same core API forward while adding Windows support, 24-bit color, unicode-aware keyboard decoding, and sequence-aware string measurement.

The library's defining choice is that it is *not* a full-screen TUI framework. Blessed gives you capability strings (`term.red`, `term.move_xy(x, y)`, `term.underline("hi")`) and a handful of context managers (`cbreak`, `raw`, `hidden_cursor`, `fullscreen`, `location`), but it does not own the event loop, the widget tree, or the screen buffer. You print sequences yourself to any file-like object. This keeps it composable — output piped to a non-tty degrades to plain text automatically — at the cost of leaving layout, diffing, and redraw logic to the caller.

The audience is authors of CLI tools, small full-screen apps, and terminal games who want ANSI styling and key handling that work identically on Windows, macOS, Linux, and BSD, and who are willing to manage their own rendering. Projects like Voltron, cursewords, Dashing, and Enlighten build on it[^1]. If you want prebuilt widgets and a managed render loop, blessed is the wrong altitude — see Alternatives.

## Getting Started

```bash
pip install blessed
```

```python
from blessed import Terminal

term = Terminal()

# Styling + centering, safe when stdout is not a tty (sequences vanish)
print(term.home + term.clear + term.move_y(term.height // 2))
print(term.black_on_darkkhaki(term.center("press any key to continue.")))

# Read a single keypress without echo or line buffering
with term.cbreak(), term.hidden_cursor():
    key = term.inkey()

print(term.move_down(2) + "You pressed " + term.bold(repr(key)))
if key.is_sequence:                 # arrow/function keys, etc.
    print("code:", key.code, "name:", key.name)
```

Blessed supports Python 3.7+ and requires no compiled extensions on Unix; on Windows it depends on `jinxed`, a pure-Python terminfo/`curses` reimplementation, since Windows ships no `curses`[^1].

## Architecture / How It Works

At construction, `Terminal()` calls `setupterm()` (via `curses` on Unix, `jinxed` on Windows) against `$TERM`, loading that terminal type's capability database. Attribute access is resolved dynamically: `term.underline` looks up the terminfo `smul`/`sgr0` pair, `term.move_xy` wraps `cup` with `tparm`, and named colors map onto the terminal's advertised color count (8, 16, 256, or 24-bit). If the output stream is not a tty, every capability returns an empty string, so the same code prints clean text when redirected to a file, pipe, or `StringIO`[^1].

Three subsystems sit on top of this capability layer:

- **Styling.** Callable "formatting strings" like `term.bold_red` compose: attributes chain (`term.bold_underline_blue`), and calling one wraps text and auto-appends the reset sequence. `term.color_rgb(r, g, b)` / `term.on_color_rgb(...)` emit truecolor sequences when the terminal claims support, otherwise degrade to the nearest indexed color.
- **Keyboard.** `inkey()` reads bytes, decodes them in the locale's preferred encoding, and resolves multi-byte escape sequences into `Keystroke` objects carrying `.code` and `.name` (e.g. `KEY_LEFT`). It supports a timeout and an "escape delay" (`esc_delay`) heuristic to disambiguate a lone Escape from the start of a sequence. This only works inside `cbreak()` or `raw()`, which put the tty into the corresponding termios mode and restore it on exit.
- **Sequence-aware text.** Because ANSI sequences occupy zero display columns, blessed provides `length()`, `strip_seqs()`, `wrap()`, `center()`, `ljust`/`rjust`, and width-correct handling of wide (CJK) and zero-width characters. This is the piece hand-rolled ANSI code usually gets wrong.

Cursor positioning is offered two ways: absolute moves (`move_xy`, `move_up`), and the `location()` context manager, which saves and restores cursor position around a block. `get_location()` actively queries the terminal (writes a Device Status Report, reads the reply) to learn where the cursor is — a round-trip that can block or mis-parse if the terminal is busy or the response is intercepted.

## Production Notes

- **`get_location()` is a synchronous terminal round-trip.** It writes an escape query and blocks reading the answer back through stdin. If another reader is consuming stdin, if input is not a tty, or if the terminal never answers, it can hang up to its timeout or return stale coordinates. Treat it as best-effort, always pass a `timeout`, and avoid it in the hot path.
- **`cbreak`/`raw` mutate global termios state.** They set and restore the tty via `termios`; an uncaught exception or a hard `os._exit` inside the block can leave the user's shell in a no-echo, no-linebuffer state. Always let the context manager exit normally, and be cautious mixing it with threads or subprocesses that also touch the tty.
- **No render loop, no diffing.** Blessed does not track what is on screen. Full-screen apps that repaint naively will flicker and burn bandwidth over SSH. You must implement your own dirty-region redraw; the library only hands you the movement and styling primitives.
- **Capabilities depend on `$TERM`.** Output is only as capable as the terminfo entry names. A wrong or minimal `$TERM` (`dumb`, a stale `xterm` on a truecolor terminal) silently downgrades color and features. Truecolor in particular is gated on the terminal advertising it; some emulators support it without saying so.
- **Windows parity is real but jinxed-mediated.** Since December 2019 Windows is a first-class target[^1], but it runs through `jinxed` rather than native curses, so edge-case capabilities can differ from Unix. Test on the actual Windows terminal you ship to (Windows Terminal vs. legacy conhost behave differently).
- **Wide/emoji width is heuristic.** Column counting uses Unicode width tables; newer emoji, ZWJ sequences, and the terminal's own rendering can disagree, throwing off `center()`/`wrap()` alignment. The kitty text-sizing protocol support (`text_sized`, `does_text_sizing`) addresses part of this only on terminals that implement it.

## When to Use / When Not

**Use when:**
- You want cross-platform ANSI colors, styles, and key input with a clean, documented API and no compiled dependency.
- You are building a CLI tool, a small full-screen app, or a terminal game and want to own the render loop.
- You need correct printable-length, wrapping, and centering of strings that already contain escape sequences.
- You want output that automatically degrades to plain text when piped.

**Avoid when:**
- You want prebuilt widgets, layout, and a managed event loop — that is a framework's job, not blessed's.
- You are building a large, stateful full-screen UI and don't want to implement your own screen diffing.
- You need mouse handling, async-native I/O, or a component/reactive model out of the box.
- You are targeting only a single platform and the standard-library `curses` already covers your needs.

## Alternatives

- textualize/rich — use instead when you want rich styling, tables, markdown, and progress bars for *output* rather than interactive full-screen control.
- textualize/textual — use instead when you want a full async TUI framework with widgets, CSS-like layout, and a managed event loop.
- urwid/urwid — use instead when you want a mature widget/loop toolkit for full-screen console UIs on Unix.
- prompt-toolkit/python-prompt-toolkit — use instead when the app is primarily a rich interactive prompt / REPL with completion and editing.
- Python standard-library `curses` — use instead when you are Unix-only and want zero third-party dependencies and don't mind the low-level API.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-03-01 | `blessed` repository created as a fork of `erikrose/blessings`[^2]. |
| 1.x | since 2014 | Ongoing 1.x line on PyPI; API-compatible superset of blessings. |
| — | 2019-12 | Windows support added via `jinxed`[^1]. |
| — | ~2021 | Python 2 support dropped; library moved to Python 3-only. |
| — | 2026-07 | Actively maintained; last push 2026-07-14, MIT-licensed[^3]. |

Exact per-release version numbers are omitted where not verified here; consult the PyPI release history and changelog for precise tags[^4].

## References

[^1]: Blessed README and introduction. https://github.com/jquast/blessed
[^2]: `erikrose/blessings`, the upstream project blessed forked from. https://github.com/erikrose/blessings
[^3]: GitHub API repository metadata for `jquast/blessed`, retrieved 2026-07-15 (1,488 stars, 82 forks, MIT, last push 2026-07-14).
[^4]: Blessed on PyPI (release history and changelog). https://pypi.org/project/blessed/
[^5]: Full documentation. https://blessed.readthedocs.io/en/latest/

## Tags

python, terminal, cli, ansi, curses, terminfo, tui, colors, keyboard-input, cross-platform, text-styling
