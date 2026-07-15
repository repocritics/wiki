# peterbrittain/asciimatics

> A single cross-platform Python API for full-screen terminal work — the same code drives ASCII animations and interactive TUIs on Linux, macOS, and Windows.

[GitHub repo](https://github.com/peterbrittain/asciimatics) ·
[Documentation](https://asciimatics.readthedocs.org/) ·
[License: Apache-2.0](https://github.com/peterbrittain/asciimatics/blob/master/LICENSE)

## Overview

Asciimatics is a Python package for building full-screen text interfaces and ASCII art animations that run identically across platforms[^1]. It has existed since 2015 and is mature and slow-moving: as of 2026 it sits at ~4.3k stars with a roughly yearly release cadence (latest tagged release 1.15.x), and commits still land on `master`, so it is maintained but not fast-evolving. Python 2 support was dropped after v1.14; current releases are Python 3 only.

The defining design choice is a single `Screen` abstraction that hides the fact that terminals are not portable. On Unix it sits on top of `curses`; on Windows it talks to the Win32 console API via `pywin32` rather than curses (which Windows lacks a real implementation of)[^1]. This is what lets the exact same script produce coloured output, keyboard/mouse input, and resize handling on all three OSes — the feature most competing libraries (urwid, blessed) do not offer.

The tension is that asciimatics serves two audiences with one engine. The lower half is a frame-based animation loop (Screens, Scenes, Effects, Renderers); the widget/TUI framework (`Frame`, `Layout`, `Widget`) is layered on top of that same loop. This makes the codebase coherent and the animation demos delightful, but means a static data-entry form is still driven by the same redraw-per-frame machinery built for particle systems — a coupling that shows up in CPU behaviour and event handling (see Production Notes).

## Getting Started

```bash
pip install asciimatics    # Windows without pip also needs pywin32
```

```python
# Low-level Screen API — coloured text at any position, works on Win/macOS/Linux.
from random import randint
from asciimatics.screen import Screen

def demo(screen):
    while True:
        screen.print_at('Hello world!',
                        randint(0, screen.width), randint(0, screen.height),
                        colour=randint(0, screen.colours - 1),
                        bg=randint(0, screen.colours - 1))
        if screen.get_key() in (ord('Q'), ord('q')):
            return
        screen.refresh()

Screen.wrapper(demo)
```

`Screen.wrapper` sets up and tears down the terminal (raw mode, alternate buffer, cursor hide) and guarantees restoration even on exception — the equivalent of curses' `wrapper`, but portable.

## Architecture / How It Works

The stack is four layers, bottom to top:

- **Screen** — the portability boundary. A double-buffered character grid; `refresh()` diffs the buffer against what is on screen and emits only the changed cells. Backed by `curses` on Unix and `pywin32` on Windows. This is the only OS-specific code; everything above is pure Python.
- **Renderers** — objects that turn something into coloured text. `StaticRenderer` holds fixed frames; `DynamicRenderer` regenerates per frame. `FigletText` wraps pyfiglet for banner fonts; `ImageFile` / `ColourImageFile` convert JPEG/GIF to ASCII (via Pillow); there are box, bar-chart, rainbow, and fire renderers.
- **Effects & Scenes** — an `Effect` draws to the Screen for a given frame number; a `Scene` is an ordered list of Effects with a duration. `Screen.play([scenes])` runs the loop, advancing frames at a fixed rate (default ~20 fps) and dispatching input events to each Effect.
- **Widgets / TUI** — `Frame` is itself an `Effect`. It hosts one or more `Layout` objects, each holding `Widget`s (`Text`, `Button`, `ListBox`, `RadioButtons`, `FileBrowser`, etc.) in a column grid. Frames follow a model-view convention: widget values sync to/from a data dictionary via `save()` / `reset()`. Navigation, focus, and tab order are handled by the Frame.

Because the widget system is an Effect inside the same `play()` loop, a TUI app is architecturally an animation that happens to contain forms. Event handling is cooperative and single-threaded: `get_event()` is non-blocking, the loop polls it each frame, and there is no built-in async runtime.

## Production Notes

- **Windows is Win32, not curses.** Behaviour differs between the legacy `conhost` console and Windows Terminal — colour depth, unicode rendering, and mouse support all vary. Test on the actual terminal your users run, not just under WSL (which is really the Linux path).
- **Resize is an exception you must catch.** When the terminal is resized, asciimatics raises `ResizeScreenError` out of `play()`. The expected pattern is a loop that catches it, rebuilds the Scenes against the new dimensions, and calls `play(..., start_scene=last_scene)` to resume. Apps that ignore this crash on resize — a very common first-time footgun.
- **The loop is always running.** Even an idle form redraws on its frame tick, so a plain asciimatics TUI is not a zero-CPU event-wait like a hand-written `select()` loop. For battery-sensitive or long-idle apps this matters; tune the frame rate or gate work behind actual input.
- **No native async.** There is no asyncio integration. Interleaving network I/O with the UI means either running the Screen loop in its own thread, or doing non-blocking work inside effect updates and keeping each frame's work short — a long call inside an update janks the whole animation.
- **CJK / wide characters** are handled via `wcwidth`, but you are still responsible for the terminal font actually containing the glyphs; width math can drift if the terminal disagrees with `wcwidth`.
- **Dependencies pull weight.** Pillow (image conversion), pyfiglet (fonts), wcwidth, and on Windows pywin32 are transitive installs; the Pillow dependency in particular is heavier than some terminal projects want.

## When to Use / When Not

**Use when:**
- You need one codebase to run a terminal UI on Windows *and* Unix without WSL or per-OS branches.
- You want ASCII art, banners, particle effects, or image-to-ASCII with almost no code.
- Your TUI is a set of forms/lists and you value a small, stable, dependency-light-enough API over cutting-edge styling.

**Avoid when:**
- You want a modern, reactive, CSS-styled TUI with async as a first-class citizen — Textual is the better fit.
- You only need a few interactive prompts, not a full screen — prompt_toolkit or questionary are lighter.
- You need a high-performance, zero-idle-CPU event loop or deep async I/O integration.

## Alternatives

- Textualize/textual — modern async, CSS-styled, reactive TUI framework; use instead when you want rich layout and are Python 3.8+ with an async app.
- urwid/urwid — mature, battle-tested Unix TUI toolkit; use when you are Unix-only and want a proven widget set with a callback/loop model.
- prompt-toolkit/python-prompt-toolkit — full-screen apps plus REPL-style prompts with async support; use for interactive prompts and line editors.
- Textualize/rich — formatted output, tables, progress bars (not an interactive full-screen event loop); use when you only need pretty printing, not input handling.
- jquast/blessed — thin, Pythonic wrapper over terminal capabilities; use when you want low-level styling/positioning control on Unix without an animation engine.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.7.0 | 2016-09-24 | Early stable line; widgets and effects established[^2]. |
| 1.10.0 | 2018-09-18 | Continued widget/effect expansion[^2]. |
| 1.11.0 | 2019-05-10 | — |
| 1.12.0 | 2020-11-15 | — |
| 1.13.0 | 2021-04-05 | — |
| 1.14.0 | 2022-04-23 | Last release to support Python 2[^1]. |
| 1.15.0 | 2023-10-25 | Current major line; Python 3 only[^2]. |
| 1.15.1 | 2024 | Patch release on the 1.15 line. |

## References

[^1]: asciimatics README — cross-platform scope, Windows `pywin32` backend, Python 2 dropped after v1.14. https://github.com/peterbrittain/asciimatics/blob/master/README.rst
[^2]: asciimatics releases and tags (dates from GitHub release metadata). https://github.com/peterbrittain/asciimatics/releases

## Tags

python, tui, terminal, curses, ascii-art, console, cross-platform, animation, text-ui, widgets
