# terkelg/prompts

> Lightweight, promise-based interactive CLI prompts for Node.js — the small alternative to Inquirer.

[GitHub repo](https://github.com/terkelg/prompts) ·
[npm](https://www.npmjs.com/package/prompts) ·
[License: MIT](https://github.com/terkelg/prompts/blob/master/license)

## Overview

`prompts` is a Node.js library for asking users questions on the command line: text, password, number, confirm, select, multiselect, autocomplete, date, and a handful of others. It was written by Terkel Gjervig and first published in 2018[^1] as a deliberate reaction to Inquirer.js, which by then had accumulated a large dependency tree and a plugin-based architecture. `prompts` ships as a single small package with only two runtime dependencies — `kleur` for colors and `sisteransi` for ANSI escape codes[^2], both tiny and by the same third-party author (lukeed).

The defining tradeoff is **size and simplicity versus feature ceiling and maintenance**. The API is intentionally flat: you pass an array (or single object) of question descriptors, each with a `type`, `name`, and `message`, and get back a plain object keyed by `name`. Everything is promise-based (`async`/`await`), prompt objects can have function-valued fields for dynamic/conditional flows, and there is a built-in `inject()` path for feeding answers programmatically in tests. This makes it a common default in scaffolding tools and small CLIs where pulling in a larger framework is unwanted.

The counterweight is maintenance. The repository has been in a low-activity state for years: as of mid-2026 it carries 152 open issues and 9.3k stars, but meaningful releases stopped landing well before that, and many bug reports and pull requests sit unmerged. It is best understood as *stable and battle-tested but effectively feature-frozen* — you adopt it for what it already does, not for what it might grow into.

## Getting Started

```bash
npm install prompts
```

```js
const prompts = require('prompts');

(async () => {
  const response = await prompts({
    type: 'number',
    name: 'age',
    message: 'How old are you?',
    validate: value => value < 18 ? 'Nightclub is 18+ only' : true
  });

  console.log(response); // => { age: 24 }
})();
```

A chain of questions with a conditional (dynamic) prompt — a `type` of `null`/falsy skips the question:

```js
const questions = [
  { type: 'text',   name: 'dish',    message: 'Favorite dish?' },
  { type: prev => prev === 'pizza' ? 'text' : null,
    name: 'topping', message: 'Name a topping' }
];
const answers = await prompts(questions);
```

## Architecture / How It Works

The core is small and readable. The exported `prompts()` function iterates the question list sequentially, resolving any function-valued properties (`type`, `message`, `initial`, `format`, etc.) just before each question by calling them with `(prev, values, prompt)` — this is how conditional and dependent prompts work without a separate DSL. Each question is delegated to a per-type prompt class (under `lib/elements/`), which extends a common base that puts the TTY into raw mode, listens for keypress events, and re-renders the line on every state change.

Rendering is done by hand with ANSI escape sequences via `sisteransi` (cursor movement, line clearing) and colored with `kleur`. There is no virtual DOM or diffing layer; each render clears the prompt's lines and re-prints. This keeps the dependency surface minimal but means multi-line prompts (multiselect with many rows, autocomplete lists) are the most fragile area for terminal-rendering bugs, especially on Windows consoles and inside non-standard TTYs.

Two escape hatches matter for real usage. `prompts.override(obj)` pre-answers any question whose `name` matches a key — designed to be fed `process.argv`/yargs so CLI flags bypass interactive input. `prompts.inject([...])` pushes answers onto an internal queue that resolves prompts immediately; a value that is an `Error` instance simulates the user cancelling. `inject` is explicitly documented as test-only, and because the queue is module-level global state, it is easy to leak answers between tests if not drained.

A subtle but important design point: when a prompt is cancelled (Esc, Ctrl+C, Ctrl+D), **no key is set on the response object at all** rather than an `undefined` value. Consumers must detect cancellation via `onCancel` or by checking for missing keys — not by reading a falsy answer.

## Production Notes

- **Maintenance risk.** This is the headline caveat. Releases have effectively stalled and the issue tracker (152 open) includes long-standing rendering and edge-case bugs. Nothing is actively broken for the common paths, but do not expect fixes or new prompt types. If you need an actively maintained library, weigh the alternatives below.
- **Types are external.** No TypeScript definitions ship in the package; you install `@types/prompts` from DefinitelyTyped separately, and those types can lag or diverge from actual runtime behavior for the more dynamic fields.
- **CommonJS-first.** The package is authored as CommonJS (`require`). It works from ESM via Node's interop (`import prompts from 'prompts'`), but there is no separate native ESM/`exports` map, which occasionally trips up strict bundler configurations.
- **Non-TTY / CI environments.** In a pipe or CI with no interactive terminal, prompts will not behave usefully. Drive it with `override` (from parsed flags) or `inject` for tests; do not rely on interactive input in automated contexts.
- **`inject` global state.** Because injected answers live in a shared module-level array, tests that inject must fully consume what they push, or leftover values bleed into the next test. Reset between cases.
- **Multiselect/autocomplete on large lists.** Rendering re-prints the visible window each keystroke; very large choice arrays and small terminal heights are where users historically report flicker, misaligned output, and slow filtering.
- **Signal handling.** The library manages raw mode and restores the terminal on exit, but if your process traps `SIGINT` itself you can end up fighting it — decide who owns Ctrl+C.

## When to Use / When Not

**Use when:**
- You want a small, dependency-light prompt library for a CLI or scaffolder.
- Your prompt flows fit the built-in types and you value a flat, promise-based API.
- You need programmatic answer injection for tests (`inject`) or flag-driven non-interactive runs (`override`).

**Avoid when:**
- You need active maintenance, new prompt types, or responsive bug fixes.
- You want first-class TypeScript types shipped in-package.
- You need rich/custom prompt widgets, nested menus, or heavy theming — a more extensible framework fits better.
- Your CLI targets a design-forward, modern terminal aesthetic — newer libraries invest more there.

## Alternatives

- SBoudrias/Inquirer.js — the original heavyweight; now modularized as `@inquirer/prompts`. Use instead when you want an actively maintained, extensible, plugin-friendly framework and don't mind more surface area.
- enquirer/enquirer — more built-in prompt types and theming than prompts, still relatively light. Use when you need richer prompts but want a single package.
- bombshell-dev/clack (`@clack/prompts`) — modern, actively developed, opinionated visual style with grouped flows. Use for new CLIs where aesthetics and current maintenance matter.
- vadimdemedes/ink — React-for-the-terminal. Use when your CLI UI is complex enough to warrant a component/render model rather than sequential questions.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-02 | First publish; single-package, promise-based alternative to Inquirer[^1]. |
| 2.x | 2019 | Second-generation API; added prompt types and `inject`/`override`. |
| 2.4.x | ~2021 | Last actively released line; Node 14+ baseline stated in README[^2]. |
| (frozen) | 2022–2026 | Repository in low-maintenance mode; 152 open issues, no new feature releases. |

Exact per-version release dates beyond the above are not confidently verified here; treat the table as approximate.

## References

[^1]: Repository created 2018-02-24; package published to npm as `prompts`. https://github.com/terkelg/prompts
[^2]: README — install, dependency notes (`kleur`, `sisteransi`), Node 14+ support, and full prompt-type / API reference. https://github.com/terkelg/prompts/blob/master/readme.md

## Tags

javascript, nodejs, cli, command-line, interactive-prompts, terminal, tui, prompt, developer-tools, scaffolding
