# vercel-labs/agent-browser

A native Rust command-line tool that gives AI agents fast, scriptable control of a real Chrome browser.

## What it is

agent-browser is a browser automation CLI built as a native Rust binary and designed for AI agents to drive. It exposes browser actions — navigate, snapshot, click, fill, read, screenshot, and more — as CLI commands and as an MCP server, so an agent can operate a page without embedding a heavier Node/Playwright stack. It talks to Chrome over the Chrome DevTools Protocol and can download Chrome from Chrome for Testing, while also detecting existing Chrome, Brave, Playwright, and Puppeteer installs; no Playwright or Node.js is required for the daemon itself.

## Key features

- Accessibility-tree snapshots that return element refs (`@e1`, `@e2`) an agent can click, fill, or read by reference, plus traditional CSS selectors and semantic locators (by role, text, label, placeholder, alt, title, or test id).
- A `read` command that fetches agent-friendly text without launching Chrome, negotiates `text/markdown`, walks ancestor paths to find the nearest `llms.txt`, and can read the active tab's rendered DOM including auth state.
- Batch execution of multiple commands in one invocation (arguments or JSON via stdin) with an optional `--bail`, avoiding per-command process startup overhead.
- First-class React introspection (component tree, fiber inspect, render recording, Suspense classifier) when launched with `--enable react-devtools`, plus framework-agnostic Web Vitals (LCP/CLS/TTFB/FCP/INP) and SPA `pushstate`.
- An MCP stdio server (`agent-browser mcp`) with tool profiles (core, network, state, debug, tabs, react, mobile, all) so clients can load a small or full typed surface, with typed fields and per-tool approval prompts.
- Session isolation, multiple authentication approaches (Chrome profile reuse, persistent profiles, session persistence, state files, an encrypted auth vault), network routing/HAR capture, cookies/storage control, and snapshot/screenshot/URL diffing.

## Tech stack

- Rust native CLI (`cli/Cargo.toml`) with a daemon architecture and a native CDP client (`cli/src/native/cdp/`); no Node.js runtime needed for the daemon.
- Distributed as an npm package (`agent-browser`, version 0.32.3) that installs the native binary, plus Homebrew and `cargo install`.
- Building from source requires Node.js 24+ and pnpm 11+ (`packageManager: pnpm@11.1.3`, `engines.node >= 24.0.0`) and Rust; native builds run `cargo build --release` and cross-compile via Docker for Linux/Windows and dual-target for macOS.
- Chrome from Chrome for Testing is the default browser channel; an optional Lightpanda engine is present (`native/cdp/lightpanda.rs`), and WebDriver backends exist for Safari, iOS, and Appium (`native/webdriver/`).
- The MCP server defaults to protocol 2025-11-25 and exchanges newline-delimited JSON-RPC over stdio; a documentation site is built with Next.js/MDX under `docs/`.
- Licensed Apache-2.0.

## When to reach for it

- You want an AI agent to control a real browser through a fast CLI or MCP server rather than a language-specific automation library.
- You need accessibility-tree snapshots with stable refs as the primary way for a model to target elements.
- You want to fetch agent-readable page/docs text (markdown, `llms.txt`) with or without launching a browser.
- You need React component introspection and Web Vitals as part of automated inspection or debugging.
- You want isolated sessions, reusable auth state, and network interception/HAR capture for repeatable agent runs.

## When *not* to reach for it

- You are scripting browser automation directly in Node, Python, or TypeScript and want an in-process API rather than a CLI/daemon.
- You need broad, long-supported cross-browser coverage; the default path is Chrome/Chromium via CDP (with optional Lightpanda and WebDriver backends).
- You want a settled, mature 1.0 tool; this is a 0.x release with a large open-issue count and frequent changes.
- Your environment cannot run Chrome for Testing or provide the required build toolchain when compiling from source.

## Maturity signal

The repository was created in January 2026 and is pushed to almost daily, so it is under very active development. It gathered large star and fork counts in only a few months, signaling strong early interest, while the high open-issue count (in the hundreds) reflects both heavy usage and a young, fast-moving codebase still working through edge cases. The npm version is 0.32.3 — pre-1.0 — and the Apache-2.0 license under the vercel-labs organization suggests an experimental but organization-backed effort. Treat it as actively maintained and rapidly evolving, with APIs and behavior that may still shift.

## Alternatives

- Playwright or Puppeteer — reach for these when you want to script browser automation directly in Node, Python, or TypeScript with an in-process API rather than an agent-oriented CLI.
- browser-use — use when you want a Python framework where an LLM drives the browser in-loop rather than a language-agnostic CLI/MCP surface.
- Stagehand — use when you want AI browser automation embedded in TypeScript application code instead of invoked as a separate command-line tool.

## Notes

- Element refs (`@e1`) and tab handles (`t1`, `t2`) are deliberately non-numeric and stable within a session so agents and scripts keep referring to the same element or tab even as others open and close; positional integers like `tab 2` are rejected.
- The `read` command will not fetch `llms-full.txt` unless explicitly asked, and clicks fail early when another element (such as a consent banner) covers the target's click point, reporting the covering element.

## Tags

rust, cli, browser-automation, cdp, chrome, mcp, ai-agents, testing
