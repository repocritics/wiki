# microsoft/playwright-mcp

A Model Context Protocol server that gives LLMs browser automation through Playwright's accessibility tree instead of screenshots.

## What it is

Playwright MCP is an MCP server that exposes browser automation to LLM clients. Instead of feeding screenshots to a vision model, it lets an agent interact with web pages through structured accessibility snapshots, which the README frames as faster and more deterministic. It is aimed at developers wiring browser control into MCP-capable tools such as VS Code, Cursor, Claude Desktop, and other clients. The README positions MCP as best for specialized agentic loops that benefit from persistent browser state and iterative reasoning over page structure.

## Key features

- Operates on Playwright's accessibility tree rather than pixel-based input, so no vision model is required.
- Deterministic tool application that avoids the ambiguity of screenshot-based approaches.
- Broad MCP client coverage with documented setup for VS Code, Cursor, Claude Code, Cline, Codex, Copilot, Gemini CLI, and many others.
- Selectable browser profiles: persistent (default), isolated per-session, or connected to a running browser via the Playwright extension.
- Extensive configuration flags and environment variables covering browser choice (chrome, firefox, webkit, msedge), headless mode, device and mobile emulation, proxies, network origin allow/block lists, secret redaction, and optional capabilities (vision, pdf, devtools).
- HTTP/SSE transport via `--port` for headless hosts or worker processes, in addition to the default stdio transport.

## Tech stack

- TypeScript, requiring Node.js `>=18`.
- Depends on playwright and playwright-core (`1.62.0-alpha` build) and, for development, `@modelcontextprotocol/sdk` (`^1.25.2`).
- Published as the npm package `@playwright/mcp` (manifest version `0.0.78`), typically run through `npx`.
- Licensed Apache-2.0.

## When to reach for it

- Exploratory automation, self-healing tests, or long-running autonomous workflows where maintaining continuous browser context matters.
- Giving an MCP-capable agent structured page interaction without relying on screenshots or vision models.
- Reusing an existing logged-in browser session through the browser extension.
- Running browser automation on a display-less host by enabling the HTTP transport.

## When *not* to reach for it

- Coding agents optimizing for token efficiency, where the README recommends the Playwright CLI with Skills instead of loading MCP tool schemas and accessibility trees.
- Any situation requiring a security boundary; the README states plainly that Playwright MCP is not one, and that file-access and secret features are conveniences rather than secure boundaries.
- Multiple concurrent clients sharing one persistent profile in the same workspace, which conflict unless each runs isolated or with a distinct user data directory.

## Maturity signal

The project reached over 35,000 stars in roughly a year, is maintained by Microsoft under Apache-2.0, was last pushed in July 2026, and carries only 7 open issues, which together indicate active maintenance and tight issue triage. The npm package remains in the `0.0.x` range, so the public interface should still be treated as pre-1.0 and subject to change.

## Alternatives

- Playwright CLI with Skills — use it when a coding agent wants concise, purpose-built commands and lower token cost than MCP tool schemas; the README recommends this path explicitly.
- Playwright (the library) — use it when you are writing browser automation directly in code rather than mediating it through an LLM and MCP.

## Notes

- The README openly steers coding-agent users toward the CLl+Skills alternative for token efficiency, an unusual amount of self-directing away from the project's own MCP surface.
- File-system access is restricted to workspace roots by default and navigation to `file://` URLs is blocked unless unrestricted access is enabled.
- Secret redaction is documented as a convenience to keep sensitive strings out of tool responses, not a guaranteed security control.

## Tags

typescript, mcp, browser-automation, playwright, testing, llm, agents, nodejs
