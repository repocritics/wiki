# modelcontextprotocol/servers

The official reference + community-maintained registry of Model Context Protocol (MCP) servers — first-party implementations for filesystem, GitHub, Slack, databases, and many more integrations.

## What it is

A monorepo of MCP servers — programs that expose tools / resources to MCP-capable AI agents (Claude, Cursor, OpenAI agents, etc.). Anthropic + community contributors publish reference implementations here for common integrations (filesystem, GitHub, GitLab, Postgres, Brave Search, Puppeteer, Slack, Google Maps, AWS, etc.). The canonical "show me an MCP server" reference.

## Key features

- 30+ first-party MCP server implementations.
- Reference code in TypeScript / Python for the major SDKs.
- Companion: separate `modelcontextprotocol/` org repos for the spec, SDK, and inspector tool.
- TypeScript primary (most servers in the official set are TS).
- License is `NOASSERTION` per SPDX — each server has its own LICENSE; verify.

## Tech stack

- TypeScript primary across the reference servers.
- MCP SDK packages from the same org.
- Distributed via npm (per-server packages).

## When to reach for it

- You're a Claude Code / Cursor / agent user who wants a vetted reference server for a common integration.
- You're writing your own MCP server and want canonical examples to copy.

## When *not* to reach for it

- You need vendor-stable servers with SLAs — these are reference / community implementations.
- The license matters — verify per-server LICENSE before commercial deployment.

## Maturity signal

87k stars, 11k forks, actively maintained. Open-issues count of 500 tracks per-server feature requests + breakage reports.

## Alternatives

- Community MCP-server collections (`punkpeye/awesome-mcp-servers` etc.).
- Writing your own server against the MCP spec.

## Tags

artificial-intelligence, large-language-model, agent, model-context-protocol, typescript, mcp, anthropic, awesome-list
