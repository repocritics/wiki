# upstash/context7

Context7 — Upstash's MCP server that gives AI coding agents access to up-to-date library documentation. Solves the "agent uses outdated API" problem.

## What it is

An MCP (Model Context Protocol) server from Upstash that exposes a `resolve-library-id` → `query-docs` flow: agents resolve a library by name, then fetch current documentation for it. The library covers thousands of popular packages with frequently-refreshed docs. Aimed at coding agents (Claude Code, Cursor, etc.) that need authoritative current docs rather than potentially-stale training-data knowledge.

## Key features

- MCP server distributing current docs for thousands of libraries.
- `resolve-library-id` + `query-docs` API surface.
- Hosted by Upstash; free for individual / small use.
- npm-installable for self-hosting.

## Tech stack

- TypeScript primary.
- MCP server protocol.

## When to reach for it

- You're a Claude Code / Cursor user and want your agent to use current API docs.
- You're standing up an agent and want to mitigate stale-training-data hallucinations.

## When *not* to reach for it

- You want self-hosted by default — verify the hosted vs. self-host story for your use case.
- The library you need isn't in Context7's catalog — fallback to your own docs.

## Maturity signal

Actively maintained under Upstash.

## Alternatives

- Custom MCP server pointing at your own docs.
- Direct doc-search via fetch / scrape.

## Tags

artificial-intelligence, agent, model-context-protocol, typescript, documentation, mcp-server, claude, cursor
