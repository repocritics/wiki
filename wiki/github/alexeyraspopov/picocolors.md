# alexeyraspopov/picocolors

> The smallest practical ANSI terminal-color library for Node — one file, no dependencies, deliberately minimal API.

[GitHub repo](https://github.com/alexeyraspopov/picocolors) ·
[License: ISC](https://github.com/alexeyraspopov/picocolors/blob/main/LICENSE)

## Overview

picocolors is a terminal string-coloring library: it wraps text in ANSI escape
sequences so console output can be colored, bolded, dimmed, or underlined. It
occupies the same niche as `chalk` but takes the opposite design stance —
where chalk is a feature-rich API surface, picocolors is a single ~100-line
`index.js` with zero dependencies whose entire reason to exist is being small
and cheap to load[^1]. The README frames this explicitly as an attempt to draw
attention to the `node_modules` size problem and to promote a
"performance-first culture."

Its relevance is out of proportion to its star count because it sits deep in
the JavaScript toolchain rather than in application code. PostCSS, SVGO,
Stylelint, and Browserslist depend on it[^1], which means picocolors is
transitively installed in a large share of frontend build pipelines. For those
tools, the install-size and cold-start cost of a coloring library is paid on
every `npm install` and every CLI invocation, so a 7 kB / sub-millisecond load
is a real win.

The defining tradeoff is API minimalism. picocolors gives you exactly the 16
standard ANSI colors, their bright and background variants, and a handful of
formatters — nothing more. There is no chaining (`pc.red.bold(x)` does not
exist), no truecolor/hex/256-color support, no tagged-template syntax, and no
color-styling of arbitrary RGB values. If you need those, picocolors is the
wrong tool and the README says as much by shipping an explicit chalk-migration
guide rather than pretending to be a drop-in replacement.

## Getting Started

```bash
npm install picocolors
```

```javascript
import pc from "picocolors"

console.log(pc.green(`How are ${pc.italic("you")} doing?`))

// Nesting is done by composing calls, not chaining:
console.log(pc.bgBlack(pc.white("status line")))

// Gate on support, or build a disabled instance explicitly:
if (pc.isColorSupported) console.log(pc.bold("colors on"))
let { red } = pc.createColors(false)   // red(x) === x, no escapes
```

## Architecture / How It Works

The whole library is one module. Each color function is built by a factory that
closes over an `open` sequence (e.g. `\x1b[31m` for red) and a `close`
sequence (`\x1b[39m`), returning `open + input + close`. When colors are
disabled, the factory returns the identity function `String(input)` instead, so
the disabled path has essentially no overhead.

The one piece of real logic is nesting correctness. A naive wrapper breaks when
you nest the same style, because the inner `close` sequence resets the outer
style prematurely. picocolors handles this with a `replaceClose` step: before
wrapping, if the input string already contains the close code, occurrences are
rewritten back to the open code so the outer color survives the inner reset.
This is why `pc.red("a " + pc.bold("b") + " c")` stays red throughout.

`isColorSupported` is computed once at import time from the environment:
`NO_COLOR` / `FORCE_COLOR`, `--no-color` / `--color` argv flags, whether stdout
is a TTY, the `TERM` value, Windows platform, and CI detection[^2]. Because it
is evaluated eagerly, changing the environment after import does not change the
result — the escape hatch is `createColors(enabled)`, which returns a fresh API
object with support forced on or off.

The package ships CommonJS and ESM entry points plus a hand-written `.d.ts`, and
targets Node.js v6+ and browsers[^1] — a low floor that constrains it to old
syntax and no runtime dependencies.

## Production Notes

- **No chaining is a real ergonomic cost.** Migrating chalk code means
  rewriting every `chalk.red.bold(x)` into `pc.red(pc.bold(x))` by hand; the
  README documents this as a manual, mechanical process[^1]. For heavy chalk
  users, budget the refactor rather than expecting a find-and-replace.
- **No truecolor / 256-color / hex.** Only the base ANSI palette exists. Tools
  that need RGB output (progress UIs, syntax highlighting, diff coloring) will
  hit this wall and must keep chalk or reach for an ANSI library with a wider
  palette.
- **Support detection is import-time and static.** If your process mutates
  `FORCE_COLOR`/`NO_COLOR` at runtime, or you pipe output conditionally, rely on
  `createColors()` per-destination instead of the module-level `pc.*`, whose
  enabled/disabled state was frozen at first import.
- **Version pinning matters less than usual.** The surface is tiny and has been
  stable for years, so transitive-dependency churn is low; picocolors is one of
  the least likely lines in a lockfile to cause a breaking bump.
- **It is a library, not a CLI framework.** No spinners, tables, prompts, or
  layout — it only emits colored strings. Pair it with separate tooling for
  anything richer.

## When to Use / When Not

**Use when:**
- You are writing a build tool, linter, or CLI where install size and startup
  time are user-visible costs.
- You only need the standard 16 ANSI colors plus basic formatters.
- You want zero dependencies and CJS+ESM+types with no configuration.

**Avoid when:**
- You need truecolor/hex/256-color, chaining, or template-literal styling —
  use chalk.
- You are already deep in chalk's advanced API and the migration cost outweighs
  the size saving.
- You need higher-level terminal UI (tables, spinners, boxes); picocolors is
  strictly string-in, string-out.

## Alternatives

- chalk/chalk — the feature-rich standard: chaining, truecolor/hex/256, template
  syntax. Use when you need expressive styling and can absorb the larger install.
- jorgebucaran/colorette — comparably tiny and fast, similar function-call API.
  Use when you prefer its interface; performance is in the same class.
- lukeed/kleur — tiny, with optional chaining via the `kleur/colors` entry. Use
  when you want a small library but still want chained styles.
- doowb/ansi-colors — chalk-like chaining API with no dependencies. Use when
  migrating chalk code that relies on chaining but you still want to drop deps.
- ai/nanocolors — the direct predecessor by the same PostCSS-adjacent authors,
  now effectively superseded by picocolors; not recommended for new code.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2021-09-27 | Repository created[^3]. |
| 1.0.x | 2021–2023 | Core API stabilized; adopted by PostCSS, SVGO, Stylelint, Browserslist[^1]. |
| 1.1.x | 2024 | Latest line; last push 2024-11-18[^3]. |

## References

[^1]: picocolors README — features, benchmarks, prior art, and chalk-migration
guide. https://github.com/alexeyraspopov/picocolors#readme
[^2]: `NO_COLOR` convention honored by picocolors. https://no-color.org/
[^3]: GitHub repository metadata (created 2021-09-27, last push 2024-11-18,
ISC license, ~1.7k stars) — fetched via GitHub API, 2026-07.
https://github.com/alexeyraspopov/picocolors

## Tags

javascript, nodejs, terminal, ansi, colors, console, cli, zero-dependency, formatting, tty
