# cypress-io/cypress

The end-to-end testing framework that pioneered "run tests inside the browser" — fast, debuggable, browser-aware.

## What it is

A TypeScript test framework + runner that executes tests directly inside a browser (rather than via WebDriver). Single integrated tool: test runner, assertions, browser launcher, test recording. The "Cypress browser" gives developers live, time-traveling debug inspection of each test step. Started as e2e-only; later added component testing (React, Vue, Angular, Svelte component-level harnesses).

## Key features

- Tests run in-browser with full DOM access.
- Time-traveling debugger — step through test commands, see DOM state at each.
- Automatic waiting (no `sleep(500)` for elements to appear).
- Component testing for React, Vue, Angular, Svelte.
- Cypress Cloud (paid) for test recording, parallelization, analytics.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Electron-based test runner UI.
- npm package: `cypress`.

## When to reach for it

- You want a developer-friendly e2e testing experience with strong debugging.
- You're testing single-browser flows (Cypress historically had single-tab limits; newer versions added cross-origin).
- You want component-level visual tests in the same framework as e2e.

## When *not* to reach for it

- You need cross-browser parallelism at scale — Playwright or Selenium handle this better.
- You need multi-tab flows or non-Electron browser drivers.
- You want Microsoft-style cross-language tests — Cypress is JS-only.

## Maturity signal

50k stars, 3.4k forks, MIT, actively maintained. 11+ years. Open-issues count of 1,232 is moderate.

## Alternatives

- Playwright (Microsoft) — increasingly the default for new e2e projects; multi-browser, multi-language.
- Selenium / WebDriver — legacy industry standard.
- TestCafe — alternative in-browser test runner.

## Tags

typescript, testing, end-to-end-testing, browser, framework, mit-license, javascript, automation
