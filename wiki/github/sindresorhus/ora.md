# sindresorhus/ora

> Node.js terminal spinner for showing progress on indeterminate CLI work.

[GitHub repo](https://github.com/sindresorhus/ora) ·
[License: MIT](https://github.com/sindresorhus/ora/blob/main/license)

## Overview

ora is a small library that draws an animated spinner and status text in a
terminal while a Node.js program does asynchronous work. Created by Sindre
Sorhus in 2016[^1], it has become the default choice for the "loading…" state
in CLI tools, install scripts, and build tooling across the JavaScript
ecosystem — it is a transitive dependency of a large share of npm CLIs rather
than something most developers reach for by name.

Its scope is deliberately narrow: one spinner, one line, start/stop plus a set
of terminal states (`succeed`, `fail`, `warn`, `info`). It does not do progress
bars, multi-line dashboards, or concurrent task trees. That minimalism is the
point, but it also means ora is frequently outgrown — the moment you need two
things spinning at once you are on a different library.

The defining tension in ora's history is packaging, not features. Since v6
(2021) it is pure ESM[^2], which split the ecosystem: projects still on
CommonJS cannot `require('ora')` and must either dynamic-`import()` it or pin to
the v5 line. This single decision generates more real-world friction than
anything the spinner itself does.

## Getting Started

```sh
npm install ora
```

```js
import ora from 'ora';

const spinner = ora('Loading unicorns').start();

try {
	await doWork();
	spinner.succeed('Done'); // green ✔, persists the line
} catch (error) {
	spinner.fail('Failed');  // red ✖
	throw error;
}
```

For promise-wrapped work there is a helper that picks `succeed`/`fail` from the
outcome:

```js
import {oraPromise} from 'ora';

await oraPromise(doWork(), {
	text: 'Loading',
	successText: 'Done',
	failText: 'Failed',
});
```

## Architecture / How It Works

ora is a thin state machine over ANSI escape codes. `.start()` sets a
`setInterval` timer that, on each tick, clears the current line, writes the next
spinner frame plus prefix/text/suffix, and advances the frame index. `.stop()`
clears the timer and the line. The terminal-state methods (`succeed`, `fail`,
etc.) stop the animation and persist a final line with a status symbol.

The frame data itself lives in a sibling package, `cli-spinners`[^3] — ora just
selects a named spinner (`dots` by default) and reads its `frames` and
`interval`. Coloring goes through `chalk`, status symbols through
`log-symbols`, and cursor hide/show through `cli-cursor`. ora is best understood
as the orchestration layer over that cluster of Sindre Sorhus micro-packages,
which is why its install pulls in a handful of small transitive deps.

Output defaults to `process.stderr`, not stdout — a deliberate choice so a
tool's real data on stdout stays clean and pipeable while the spinner animates
on the error stream. Rendering is gated by `isEnabled`, which auto-detects
whether the stream is an interactive TTY and whether it is running in CI; in a
non-TTY or CI context the animation is suppressed and text is emitted plainly.

Two platform-specific behaviors matter. On Windows outside Windows Terminal, ora
forces the ASCII `line` spinner because the classic console lacks reliable
Unicode support. And `discardStdin` (on by default) puts stdin into raw mode to
stop keypresses from smearing the spinner line — which changes `Ctrl+C`
handling (see below).

## Production Notes

**ESM-only since v6 (2021).** This is the single biggest operational fact. A
CommonJS codebase cannot `require('ora@6+')`; it must use `await import('ora')`
or stay on the v5.x line, which still works but no longer gets features[^2].
Many "ora suddenly broke my build" reports trace to a major-version bump across
this boundary.

**The spinner freezes on synchronous work.** JavaScript is single-threaded and
the animation runs on a timer, so any long synchronous call (a big JSON parse, a
sync crypto op, a blocking `execSync`) stops the frames dead until it returns.
The fix is to keep the event loop free — use async APIs, workers, or child
processes. There is no way around this within a single thread.

**One spinner at a time — by design.** Running two `ora` instances concurrently
produces interleaved, corrupted output. For multiple or nested progress
indicators you need a different tool (see Alternatives). ora will not warn you;
it simply renders garbage.

**CI and non-TTY output is different, and that is intentional.** Under CI or
when piped, `isEnabled` is false: no animation, no cursor hiding, colors may be
stripped, and lines are emitted plainly. This is the correct default but
surprises people who expect the same output in logs as on their terminal.

**Ctrl+C latency with raw mode.** Because `discardStdin` puts stdin in raw mode,
the terminal no longer generates `SIGINT` directly; ora re-emits `Ctrl+C` from
stdin, but if your code blocks the event loop that signal is delayed until the
block ends. If interrupt responsiveness matters and you do sync work, set
`discardStdin: false`.

**stderr redirection hides the spinner.** Since output defaults to stderr,
redirecting `2>/dev/null` or capturing only stdout will make the spinner (and
its text) vanish — occasionally a debugging surprise. Pass
`stream: process.stdout` if you need it on stdout.

## When to Use / When Not

**Use when:**
- You have a single indeterminate async task and want a clean "loading" line.
- You want sensible TTY/CI auto-detection without wiring it yourself.
- You want the ecosystem-standard spinner other tools already assume.

**Avoid when:**
- You need multiple concurrent or nested progress indicators.
- Your work is CPU-bound and synchronous (the animation will stall anyway).
- You are on CommonJS and cannot adopt ESM or dynamic import — pin v5 or pick a
  CJS-friendly alternative.
- You need a determinate progress *bar* (known total) rather than a spinner.

## Alternatives

- sindresorhus/yocto-spinner — smaller, near-zero-dependency spinner by the same
  author; use it when install size and a minimal API are the priority.
- listr2/listr2 — use when you need concurrent and nested task lists with
  per-task spinners and status.
- jcarpanelli/spinnies — use when you specifically need multiple independent
  spinners animating at once.
- npkgz/cli-progress — use when the work has a known total and you want a
  determinate progress bar instead of an indeterminate spinner.
- SamVerschueren/listr — the deprecated predecessor to listr2; avoid for new
  projects, note it only when maintaining old code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release[^1]. |
| 4.0 | 2019-08 | Last widely-used CommonJS-era line before v5. |
| 5.0 | 2020-08-07 | Final major line supporting CommonJS `require`. |
| 6.0 | 2021-08-23 | Pure ESM; drops CommonJS[^2]. |
| 7.0 | 2023-07-28 | Node maintenance bump, dependency updates. |
| 8.0 | 2023-12-22 | API and dependency refresh. |
| 9.0 | 2025-09-16 | Current major line. |
| 9.4.1 | 2026-06-22 | Latest release at time of writing. |

## References

[^1]: sindresorhus/ora — repository created 2016-03-03. https://github.com/sindresorhus/ora
[^2]: ora v6.0.0 release (ESM-only). https://github.com/sindresorhus/ora/releases/tag/v6.0.0 · Background: Sindre Sorhus, "Pure ESM package." https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c
[^3]: sindresorhus/cli-spinners — spinner frame definitions consumed by ora. https://github.com/sindresorhus/cli-spinners

## Tags

javascript, nodejs, cli, terminal, spinner, progress-indicator, tty, esm, ansi, developer-tools
