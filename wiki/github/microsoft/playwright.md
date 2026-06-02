# microsoft/playwright

Microsoft's web automation + testing framework — Chromium, Firefox, WebKit with a single API. The modern default for cross-browser e2e testing.

## What it is

A TypeScript framework + test runner that drives Chromium, Firefox, and WebKit via dedicated browser-control protocols (rather than WebDriver). Authored at Microsoft by ex-Puppeteer engineers. Provides auto-wait, network interception, screenshot/video recording, and cross-browser-parallel test execution. Has displaced Cypress and Selenium as the default for new web-testing projects.

## Key features

- Cross-browser: Chromium, Firefox, WebKit — same API for all three.
- Auto-wait: assertions wait for elements without manual `sleep`.
- Network mocking + interception.
- Screenshots + video + trace files per test.
- Multi-language: TypeScript, JavaScript, Python, Java, .NET.
- VS Code extension with codegen + test debugging.
- Parallel test execution + cross-machine sharding.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary; bindings for JS, Python, Java, .NET.
- Per-browser control protocols (CDP for Chromium, custom for Firefox + WebKit).

## When to reach for it

- You're starting new e2e tests in 2026 — Playwright is the modern default.
- You need true cross-browser testing (Firefox + WebKit + Chromium).
- You want multi-language support — Cypress is JS-only.

## When *not* to reach for it

- You're maintaining a Cypress test suite — migration is significant work.
- You want a browser-in-browser experience like Cypress's runner — Playwright's UI is more headless-flavored.

## Maturity signal

90k stars, 6k forks, Apache 2.0, actively maintained by Microsoft. Open-issues count of 164 is exceptionally low — tight team triage. 5+ years.

## Alternatives

- Cypress — older browser-in-browser alternative.
- Selenium / WebDriver — legacy industry standard.
- Puppeteer — Chromium-only predecessor.

## Tags

typescript, testing, end-to-end-testing, browser, framework, microsoft, apache-license, chromium, firefox, webkit, automation
