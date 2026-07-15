# SBoudrias/Inquirer.js

> A collection of interactive command-line prompt components for Node.js — the default way JavaScript CLIs ask questions.

[GitHub repo](https://github.com/SBoudrias/Inquirer.js) ·
[License: MIT](https://github.com/SBoudrias/Inquirer.js/blob/main/LICENSE)

## Overview

Inquirer is a set of interactive terminal prompts — text input, single- and multi-select lists, confirms, password fields, autocomplete search, and more — for building command-line wizards in Node.js. It was written by Simon Boudrias (also the maintainer of Yeoman), first published in 2013, and for roughly a decade was the de facto prompt layer for the JavaScript CLI ecosystem: Yeoman generators, `npm init` flows, framework scaffolders, and countless internal tools depend on it either directly or transitively[^1].

The defining fact about Inquirer today is that there are **two Inquirers**. The original monolithic `inquirer` package used a single `inquirer.prompt([...])` call taking an array of question objects, and was built internally on RxJS observables. Around 2023 the project was rewritten from the ground up into a monorepo of small scoped packages under `@inquirer/*`, with a completely different API: each prompt is now a standalone async function you `await`[^2]. The rewrite dropped RxJS, shrank install size, and improved startup time, at the cost of breaking the old question-array API and leaving behind a large ecosystem of community prompt plugins that targeted the legacy package.

Both packages are still published. The new `@inquirer/prompts` is where active development happens; the legacy `inquirer` package continues to receive version bumps (14.x as of 2026) but is described by the maintainer as maintained-not-developed[^2]. New projects are pointed at `@inquirer/prompts`; existing projects on the old array API are not forced to migrate but get no new features.

## Getting Started

```sh
npm install @inquirer/prompts
```

```js
// Modern API — one function per prompt, each awaited
import { input, select, confirm } from '@inquirer/prompts';

const name = await input({ message: 'Project name?' });

const framework = await select({
  message: 'Pick a framework',
  choices: [
    { name: 'Vite', value: 'vite' },
    { name: 'Next.js', value: 'next' },
    { name: 'None', value: null },
  ],
});

const proceed = await confirm({ message: `Scaffold ${name}?` });
```

Answers are composed by hand (plain variables or an object), not returned as a single merged hash the way the legacy `inquirer.prompt([...])` did.

## Architecture / How It Works

The current codebase is a pnpm/Turbo monorepo. The pieces that matter:

- **`@inquirer/prompts`** — the umbrella package re-exporting every built-in prompt (`input`, `select`, `checkbox`, `confirm`, `search`, `password`, `expand`, `editor`, `number`, `rawlist`). Each prompt also ships as its own package (`@inquirer/input`, `@inquirer/select`, …) so you can depend on just one.
- **`@inquirer/core`** — the runtime every prompt is built on. It exposes a **React-style hooks API** (`useState`, `useEffect`, `useRef`, `useKeypress`, `usePrefix`, `useMemo`) inside a `createPrompt` factory. A prompt is a render function that returns the string to draw; hooks re-run it on each keypress and the core diffs and repaints the terminal[^3].
- **`@inquirer/testing`** — a harness that renders a prompt to an in-memory stream so you can assert on frames and simulate keystrokes without a real TTY.

Every prompt is a function of two arguments: the prompt config (message, choices, validation) and an optional runtime **context** — `input` / `output` streams (default `process.stdin` / `process.stdout`), `clearPromptOnDone`, and an `AbortSignal` for cancellation[^4]. The `AbortSignal` support is the clean way to add timeouts or wire prompts into a larger cancellation tree; aborting rejects the prompt promise with an `AbortPromptError`.

Rendering is line-based redraw over ANSI escape codes, not a full-screen alternate-buffer TUI. Inquirer owns stdin in raw mode for the duration of a prompt and restores it after. This is why it composes with plain `console.log` output but does not coexist automatically with an already-running `node:readline` loop — you must pause the readline interface, run the prompt, then resume.

## Production Notes

**Raw mode and piped stdin.** The single most common breakage: when you supply a custom input stream or shadow `process.stdin`, Node can fail to enter raw mode, and arrow keys / interactivity silently stop working. The fix is to call `process.stdin.setRawMode(true)` yourself before invoking a prompt[^4]. Node normally does this automatically, but loses track when stdin ownership is handed around.

**Non-interactive shells.** Inquirer cannot run without a TTY. Git hooks (husky, lint-staged), CI steps, and `npx nodemon` frequently run in non-interactive shells where keypresses never reach the process. Workarounds are environment-specific: `my-script < /dev/tty` (or `exec < /dev/tty` in a shell script) to attach a terminal, and `--no-stdin` when running under nodemon[^4]. Code that prompts unconditionally will hang or throw in these contexts — gate prompts behind a `process.stdout.isTTY` check for scriptable tools.

**Ctrl+C handling.** Pressing ctrl+c rejects the prompt promise with an `ExitPromptError` rather than killing the process, so your teardown can run. Unhandled, this prints an ugly stack trace to the user's terminal. Wrap prompt code in `try/catch`, or add a global `uncaughtException` listener that silences errors whose `name === 'ExitPromptError'`[^4]. Nearly every non-trivial CLI needs one of these.

**The two-package migration.** Moving from legacy `inquirer` to `@inquirer/prompts` is not a version bump — it is an API rewrite. The array-of-questions object, the `when`/`filter`/`transformer` question fields, and the RxJS-based `inquirer.prompt` streaming are gone. Third-party prompt types written against the old plugin interface (autocomplete, datepicker, table, tree variants) do not load in the new core and must be rewritten as `@inquirer/core` prompts. If you rely on a community prompt that was never ported, staying on legacy `inquirer` is a legitimate choice — it still works, it just won't grow.

**ESM and dual publish.** The scoped packages ship both ESM and CommonJS builds. In CJS you `await` the same functions from an async context; there is no default synchronous entry point. Bundlers benefit from the per-prompt packages for tree-shaking, but importing the umbrella `@inquirer/prompts` pulls the full set.

**Localization.** `@inquirer/i18n` is a drop-in replacement for `@inquirer/prompts` that auto-detects locale from `LANG`/`LC_ALL`/etc. and ships a handful of built-in languages; custom locales are registerable. This is separate from the core package, so English-only builds don't pay for it.

## When to Use / When Not

**Use when:**
- You're building a Node.js CLI wizard and want batteries-included prompts (validation, filtering, multi-select, search) without hand-rolling ANSI.
- You want prompts as composable `await`-able functions that slot into normal control flow.
- You need to build a custom prompt type and want a hooks model plus a real testing harness.

**Avoid when:**
- You're building a full-screen, multi-pane, continuously-updating terminal UI — that's a TUI framework's job (Ink, blessed), not a prompt library.
- You're targeting non-Node runtimes, or shipping a single static binary where a Node dependency is unwelcome.
- Your flow must run headless / non-interactively by default — prompts fight CI and hooks unless carefully guarded.
- You depend on a legacy community prompt plugin that was never ported and can't move to the new core.

## Alternatives

- clack/clack (`@clack/prompts`) — smaller, opinionated, visually polished prompt set; use when you want a modern minimal look and don't need Inquirer's breadth or custom-prompt API.
- terkelg/prompts — lightweight, dependency-light single-package prompts; use when bundle size and simplicity matter more than an extensible core.
- enquirer — feature-rich prompts (was briefly Yeoman's default); use when you want many prompt types in one package with heavy theming.
- google/zx or webpro/tasuku — use when your "prompts" are really scripting glue and you'd rather orchestrate shell than build a wizard.
- vadimdemedes/ink — use instead when you actually need a React-rendered full-screen terminal UI, not a linear question flow.

## History

| Version | Date | Notes |
|---------|------|-------|
| `inquirer` 0.x | 2013-05 | First publish; grew into the standard JS CLI prompt library (RxJS-based). |
| `inquirer` 1.0 | 2016-04 | First stable major of the legacy package[^5]. |
| `inquirer` 9.0 | 2022-06 | Last major of the classic ESM/RxJS line before the rewrite era[^5]. |
| `@inquirer/prompts` 1.0 | 2023-04 | Ground-up rewrite: per-prompt async functions on `@inquirer/core` hooks, RxJS dropped[^6]. |
| `@inquirer/prompts` 3.0 | 2023-07 | Continued API and prompt consolidation[^6]. |
| `inquirer` 10.0 | 2024-07 | Legacy package realigned onto the new foundation while keeping back-compat[^5]. |
| `@inquirer/prompts` 7.0 | 2024-10 | Node baseline bump; ongoing scoped-package majors[^6]. |

## References

[^1]: Inquirer.js repository and README — SBoudrias/Inquirer.js. https://github.com/SBoudrias/Inquirer.js
[^2]: README note on the ground-up rewrite and the still-maintained legacy `inquirer` package. https://github.com/SBoudrias/Inquirer.js#readme
[^3]: `@inquirer/core` — hooks-based prompt-building API (`createPrompt`, `useState`, `useKeypress`). https://github.com/SBoudrias/Inquirer.js/tree/main/packages/core
[^4]: README "Advanced usage" and "Recipes": context options (`input`/`output`/`signal`), raw mode, non-interactive shells, ctrl+c / `ExitPromptError`. https://github.com/SBoudrias/Inquirer.js#readme
[^5]: `inquirer` version history on npm. https://www.npmjs.com/package/inquirer?activeTab=versions
[^6]: `@inquirer/prompts` version history on npm (1.0.0 published 2023-04-24). https://www.npmjs.com/package/@inquirer/prompts?activeTab=versions

## Tags

typescript, javascript, nodejs, cli, command-line, interactive-prompts, terminal, prompt, developer-tools, tui
