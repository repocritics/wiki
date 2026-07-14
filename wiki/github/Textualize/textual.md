# Textualize/textual

> A Python framework for building full-screen terminal user interfaces (and the same app served to a browser) with a CSS-styled widget tree.

[GitHub repo](https://github.com/Textualize/textual) ·
[Official website](https://textual.textualize.io/) ·
[License: MIT](https://github.com/Textualize/textual/blob/main/LICENSE)

## Overview

Textual is a TUI (terminal user interface) framework for Python, created by Will McGugan — also the author of [Rich](https://github.com/Textualize/rich), the terminal-formatting library Textual originally built on[^1]. Where Rich renders formatted output, Textual is a full application framework: an async event loop, a DOM-like tree of widgets, a CSS-style layout and theming system, message passing, and reactive attributes. The pitch is that you write desktop-app-shaped code in plain Python and it runs in any terminal, with no GUI toolkit and no compilation step.

The distinctive bet — encoded in the current tagline "the lean application framework for Python" and the topics `tui`, `cli`, `terminal` — is that the same codebase can also be served to a web browser via `textual serve`, turning a terminal app into a shareable web app without a rewrite[^2]. This dual target is the project's most interesting idea and also the source of most of its caveats: a terminal is a constrained rendering surface (character grid, variable color support, inconsistent input), and everything Textual does is shaped by working within it.

Textual spent a long, fast-moving 0.x period (2021–2024) with frequent breaking changes before reaching a 1.0.0 stable release in late 2024[^3]. In early 2025 Will McGugan announced that Textualize, the company he had founded to commercialize the ecosystem, was winding down, while the open-source project would continue development[^4]. Treat any claim about the commercial roadmap as historical; the framework itself remains actively maintained (last push 2026-07-11).

## Getting Started

```bash
pip install textual textual-dev
```

```python
# clock.py — run with: python clock.py
from datetime import datetime
from textual.app import App, ComposeResult
from textual.widgets import Digits

class ClockApp(App):
    CSS = """
    Screen { align: center middle; }
    Digits { width: auto; }
    """

    def compose(self) -> ComposeResult:
        yield Digits("")

    def on_ready(self) -> None:
        self.update_clock()
        self.set_interval(1, self.update_clock)

    def update_clock(self) -> None:
        self.query_one(Digits).update(f"{datetime.now():%T}")

if __name__ == "__main__":
    ClockApp().run()
```

`python -m textual` runs a built-in demo; `textual serve "python clock.py"` serves the same app to a browser over a local web server.

## Architecture / How It Works

A Textual app is a tree of `Widget` objects assembled in each widget's `compose()` method — structurally a DOM. Styling is done in **Textual CSS (TCSS)**, a CSS-inspired dialect applied via a `CSS` class attribute or an external `.tcss` file[^5]. Selectors, pseudo-classes, and a query API (`query_one`, `query`) mirror the web mental model, but TCSS is a subset with terminal-specific units (cells, fractional `fr` units) and its own box model — it is not real CSS and does not accept arbitrary web CSS.

The runtime is `asyncio`-based. Widgets communicate through **messages** (an event/message queue), not direct method calls, which decouples components — a `Button.Pressed` message bubbles up the DOM to be handled by an ancestor. **Reactive attributes** (`reactive()`) declare state that, when mutated, automatically schedules a re-render and can fire watch/validate callbacks. This reactive + message model is the framework's core abstraction and the thing that makes larger apps maintainable.

Rendering goes through a **compositor**: each widget produces Rich renderables / line segments, the compositor arranges them into a screen buffer, diffs against the previous frame, and emits only the changed ANSI escape sequences to the terminal. Input (keys, mouse, resize, focus) is parsed from the terminal's escape stream back into messages. For long-running work, Textual provides **workers** (`@work`, `run_worker`) so blocking or async tasks run off the render path.

The web story reuses this same pipeline. `textual serve` (and the separate `textual-web` project) run the app process on a server and stream its terminal protocol over a websocket to an `xterm.js`-style terminal emulator in the browser — the app is unchanged; only the transport differs.

## Production Notes

- **Terminal fidelity varies.** Truecolor vs 256-color, differing emoji/Unicode width handling, and emulator-specific quirks (Windows Terminal, iTerm2, tmux, VS Code's integrated terminal, plain SSH) mean the same app can look different or mis-align across environments. Test on the terminals your users actually run.
- **Don't block the event loop.** A synchronous long call (a slow HTTP request, heavy computation, `time.sleep`) freezes the entire UI because rendering and input share the asyncio loop. Use `@work(thread=True)` or async workers for anything slow. This is the most common beginner footgun.
- **Large data views are the perf ceiling.** `DataTable` and long scrollable content can get expensive because rendering is line-oriented and re-composition has real cost. Very large tables usually need virtualization strategies or paging rather than dumping everything into the widget.
- **0.x churn.** Because the pre-1.0 series moved fast and broke APIs often, a large fraction of blog posts and Stack Overflow answers reference removed or renamed APIs. Pin the version and prefer the official docs, which track the current release. 1.0+ is far more stable but the ecosystem's written history is not.
- **Testing exists and is good — use it.** The `Pilot` harness drives an app programmatically (press keys, click, await events), and `pytest-textual-snapshot` captures SVG snapshots of the rendered screen for regression tests. Snapshot tests are the practical way to catch layout drift.
- **Web serving is not a full web framework.** `textual serve` runs a live app process per session on your server; it is excellent for sharing internal tools but is stateful, holds a connection, and does not give you a stateless HTTP app, SEO, or client-side routing. Size servers for concurrent sessions, not requests.
- **`textual-dev` is a separate install.** The dev console (which lets you see logs/prints from an app that itself owns the terminal) and live CSS editing live in the `textual-dev` package, not the core runtime.

## When to Use / When Not

**Use when:**
- You want a rich, interactive terminal app (dashboards, TUIs, installers, dev tools) in Python without a GUI toolkit.
- You already think in web terms — DOM, CSS, reactive state — and want that model in the terminal.
- You want the option to serve the same tool to a browser for teammates without building a separate web app.
- You value testability: `Pilot` + snapshot testing make TUIs unusually verifiable.

**Avoid when:**
- You need a real desktop GUI with native widgets, custom pixels, or complex graphics — a terminal grid cannot express that (use Qt, GTK, or a web stack).
- You only need styled/printed output or a progress bar, not an interactive app — Rich alone is lighter.
- You're shipping to end users on terminals you can't control and need pixel-identical rendering everywhere.
- Your app is CPU-heavy in the render loop; the line-based compositor has limits for very large or high-frequency views.

## Alternatives

- Textualize/rich — same author, lower level. Use it when you only need formatted output, tables, and progress, not a full interactive app.
- prompt-toolkit/python-prompt-toolkit — mature full-screen TUI and REPL toolkit. Use it for advanced line editors, custom key handling, or when you want lower-level control than Textual's DOM.
- urwid/urwid — long-established Python TUI library. Use it for a battle-tested, minimal-dependency stack where you don't want a CSS/reactive layer.
- charmbracelet/bubbletea — Go, Elm-architecture TUI framework. Use it when you're in the Go ecosystem and want a mature TUI story.
- ratatui/ratatui — Rust immediate-mode TUI. Use it when you need Rust-level performance for large or high-frequency terminal UIs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021 | Initial public release; built on Rich, by Will McGugan[^1]. |
| 0.x series | 2021–2024 | Long pre-stable run: TCSS styling, reactive attributes, workers, command palette, and `textual serve` added incrementally, with frequent breaking changes[^5]. |
| 1.0.0 | 2024-12 | First stable release; API stability commitment[^3]. |
| — | 2025 | Textualize (the company) winds down; open-source project continues[^4]. |

## References

[^1]: Textual documentation and Rich project, both by Will McGugan / Textualize. https://github.com/Textualize/rich
[^2]: README, "Textual ❤️ Web" and `textual serve`. https://github.com/Textualize/textual
[^3]: Textual releases / changelog. https://github.com/Textualize/textual/releases
[^4]: Will McGugan, blog post announcing the wind-down of Textualize the company while Textual continues (early 2025). https://textual.textualize.io/blog/ — treat exact dates as approximate.
[^5]: Textual guide, "CSS" and "Styles". https://textual.textualize.io/guide/CSS/

## Tags

python, tui, terminal, cli, framework, textual, rich, async, reactive-ui, cross-platform, web
