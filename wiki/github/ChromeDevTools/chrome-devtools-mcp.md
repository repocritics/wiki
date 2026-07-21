# ChromeDevTools/chrome-devtools-mcp

A Model Context Protocol server that lets coding agents control and inspect a live Chrome browser through Chrome DevTools and Puppeteer.

## What it is

`chrome-devtools-mcp` gives an AI coding agent — such as Antigravity, Claude, Cursor, or Copilot — access to a running Chrome instance for automation, debugging, and performance analysis. It acts as an MCP server that exposes DevTools capabilities: recording performance traces, inspecting network requests, reading console messages with source-mapped stack traces, taking screenshots, and capturing heap snapshots. It is aimed at developers whose agents need to verify real browser behavior rather than only generate code. A CLI is also provided for use without MCP.

## Key features

- Performance insights by recording DevTools traces and extracting actionable findings, optionally augmented with CrUX field data.
- Browser debugging: network request analysis, console messages with source-mapped stack traces, and screenshots.
- Puppeteer-based automation that waits automatically for action results.
- A large tool set grouped into input automation, navigation, emulation, performance, network, debugging, memory (heap snapshots), extensions, and WebMCP; a `--slim` mode exposes just three tools for basic tasks.
- Extensive connection and configuration flags: `--headless`, `--isolated`, `--browser-url`/`--ws-endpoint`, URL allow/block patterns, and screenshot format and size overrides.
- Ships as an agent plugin bundling the MCP server together with skills.

## Tech stack

- TypeScript, ESM output; Node engines `^20.19.0 || ^22.12.0 || >=23`.
- `puppeteer` 25.3.0, `@modelcontextprotocol/sdk` 1.29.0, `lighthouse` 13.4.0, `yargs`, and `chrome-devtools-frontend`.
- Built with `tsc` plus a rollup bundle step; tested with the Node test runner; linted and formatted with ESLint (including a rule enforcing zod schemas) and Prettier.
- Apache-2.0 licensed, authored by Google, published to npm as `chrome-devtools-mcp` at version 1.6.0.

## When to reach for it

- Your agent needs to test, debug, or measure a real web page rather than reason about code in the abstract.
- You want performance profiling, network inspection, or console/error analysis driven from an MCP client.
- You want reliable browser automation (clicks, form fills, navigation) that waits for results.
- You want a CLI to script Chrome DevTools actions without wiring up MCP.

## When *not* to reach for it

- You need a non-Chromium browser; the project officially supports only Google Chrome and Chrome for Testing.
- You are handling sensitive data; the server exposes browser content to the MCP client, and both usage statistics and performance trace URLs (to the CrUX API) are sent by default unless disabled.
- You want a lightweight headless HTTP scraper without launching a real browser.
- You are on an unsupported Node version outside the declared engine range.

## Maturity signal

The project was created in September 2025, is at version 1.6.0, is Apache-2.0 licensed and authored by Google, and shows heavy recent activity (last push mid-2026). It is young in absolute terms, but the backing of the Chrome DevTools team, a release-please automated release pipeline, generated tool and CLI docs, and a broad tool surface point to a well-resourced, actively maintained project rather than a hobby effort. Treat it as new but actively supported by its vendor.

## Alternatives

- Playwright MCP — use it instead when you need cross-browser automation across Firefox and WebKit, not just Chrome.
- Puppeteer (the library) — use it instead when you are scripting browser automation directly rather than driving it through an agent or MCP.
- Selenium / WebDriver — use it instead when you need language-agnostic, standards-based browser automation.

## Notes

- Usage statistics collection is enabled by default and is independent of Chrome's own metrics; it is disabled automatically when `CI` or the tool's opt-out environment variable is set.
- The tool count is unusually large for an MCP server, with a dedicated 12-tool memory group for heap-snapshot analysis and experimental WebMCP tools that require a very recent Chrome build with a feature flag.
- The package periodically checks npm for updates and logs a notice unless the update-check environment variable is set.

## Tags

typescript, mcp, mcp-server, browser-automation, chrome, devtools, puppeteer, debugging, performance, testing, cli
