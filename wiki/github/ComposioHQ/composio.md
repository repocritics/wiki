# ComposioHQ/composio

An SDK that gives AI agents 1000+ pre-authenticated toolkits with per-user sessions, authentication, triggers, and a sandbox.

## What it is

Composio is a TypeScript and Python SDK monorepo for equipping AI agents with real-world tools. Instead of hand-wiring integrations and OAuth for each app, you create a per-user session, hand its tools to your agent, and let the agent search, authenticate, and execute across 1000+ apps. It is aimed at developers building agents who need managed authentication, context management, and tool execution rather than building that plumbing themselves.

## Key features

- 1000+ pre-authenticated toolkits exposed through per-user sessions, with authentication and triggers handled by Composio.
- Meta tools that discover, authenticate, and execute app tools at runtime, so you don't load hundreds of tool definitions into the model's context; sessions are resumable via `composio.use()`.
- Provider adapters that map Composio tools into many agent frameworks' native tool formats, including OpenAI, OpenAI Agents, Anthropic, Claude Agent SDK, Vercel AI SDK, Google GenAI/ADK, LangChain, LangGraph, LlamaIndex, Mastra, Cloudflare, CrewAI, and AutoGen.
- A hosted MCP endpoint per session (`mcp: true`) so Claude, Cursor, or any MCP client can connect without an adapter.
- A standalone `composio` CLI to `search`, `execute`, `link` accounts, and `run` TypeScript workflows from the shell, giving coding agents a local tool surface.

## Tech stack

- TypeScript SDK (`@composio/core`) tested against Node 22+, and a Python SDK (`composio`) supporting Python 3.10+.
- Monorepo managed with pnpm (`pnpm@11.8.0`) and Turborepo; Changesets for release, Vitest for tests, ESLint, Prettier, and Husky.
- Python workspace managed with uv (`[tool.uv.workspace]`) and Ruff for lint/format; Zod used on the TypeScript side.
- Root `devEngines` warn-pins Node `>=24.17.0 <25`; MIT-licensed; default branch is `next`; root package version `0.10.0-alpha.1`.
- Additional published packages include `@composio/slim` (same API, smaller install) and `@composio/json-schema-to-zod`.

## When to reach for it

- Giving an agent authenticated access to many third-party apps without building per-app OAuth and token handling.
- Keeping tool context small by discovering and loading tools at runtime instead of injecting hundreds of definitions.
- Connecting the same tool layer to whichever framework you use (OpenAI Agents, Claude Agent SDK, LangChain, and others) via a provider adapter.
- Exposing tools to an MCP client (Claude, Cursor) through a hosted endpoint rather than embedding an SDK.

## When *not* to reach for it

- You only need one or two integrations and would rather self-host a direct MCP server or hand-roll the calls than depend on a hosted service and API key.
- You require a fully offline or self-contained stack; Composio's sessions, auth, and hosted MCP rely on its platform.
- You want a no-code automation product for non-developers rather than an SDK to embed in agent code.

## Maturity signal

Composio has a large following and a continuously active codebase, with commits pushed the same week as this writing and extensive CI across both SDKs. That said, the root package is still versioned `0.10.0-alpha.1` and the default branch is `next`, so the current line is explicitly pre-release even at this level of adoption; expect APIs to keep moving. The MIT license and dual TypeScript/Python coverage make it approachable, and the breadth of provider adapters signals real investment in ecosystem fit.

## Alternatives

- LangChain tool integrations — use instead when you want in-process tool wrappers within one framework and don't need Composio's hosted authentication and session management.
- Nango — use instead when your main need is managed OAuth and unified integration auth across many APIs, without the agent-specific tool-execution layer.
- Zapier — use instead when a no-code automation platform fits better than an SDK embedded in agent code.

## Notes

- `@composio/core` intentionally ships its TypeScript source and SDK docs so the installed package is inspectable to coding agents; `@composio/slim` offers the same API without the bundled source for a smaller install.
- The design goal of the runtime meta tools is to avoid loading large tool catalogs into the LLM context, addressing a common cost and reliability problem for tool-heavy agents.
- The repository is heavily oriented around agent-assisted development, with numerous `.agents/skills` for SDK, provider, release, and testing workflows.

## Tags

typescript, python, ai-agents, function-calling, mcp, sdk, developer-tools, llm, tool-calling, integrations
