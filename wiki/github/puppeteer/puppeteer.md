# puppeteer/puppeteer

The original headless-Chrome control library — Node.js JavaScript API for Chrome and Firefox. The predecessor to Playwright (same lead authors).

## What it is

A TypeScript / Node.js library that drives a headless (or headful) Chromium / Chrome / Firefox browser via the DevTools Protocol. Predates Playwright; Playwright was built by the same engineers after they left Google for Microsoft. Still widely used for web scraping, PDF generation, screenshot capture, and pre-rendering — its sweet spot is where Playwright's multi-browser model is overkill.

## Key features

- Chrome / Chromium / Firefox automation via DevTools Protocol.
- Page-level API: navigate, click, type, evaluate, screenshot, PDF.
- Network interception + request modification.
- Headless or headful mode.
- Lighter than Playwright when single-browser is enough.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.
- Node.js runtime.
- DevTools Protocol over WebSocket.

## When to reach for it

- You're scraping or automating Chrome / Chromium and don't need cross-browser parallelism.
- You want server-side PDF or screenshot generation.
- You want a smaller dependency than Playwright with a similar API.

## When *not* to reach for it

- You need cross-browser parallel testing — Playwright is the modern choice.
- You want full e2e test runner — Puppeteer is library-style, Playwright/Cypress ship test runners.

## Maturity signal

94k stars, 9k forks, Apache 2.0, actively maintained. 8+ years.

## Alternatives

- `microsoft/playwright` — successor with cross-browser support.
- `cypress-io/cypress` — e2e-focused framework.
- Selenium WebDriver — legacy industry standard.

## Tags

typescript, automation, browser, chromium, headless, library, apache-license, testing, scraping
