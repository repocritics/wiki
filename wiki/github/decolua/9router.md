# decolua/9router

A local OpenAI-compatible router that connects AI coding tools to many model providers with tiered auto-fallback and tool-output token compression.

## What it is

9Router is a self-hosted gateway that sits between AI coding CLIs (such as Claude Code, Codex, Cursor, Cline, and others) and 40-plus model providers. It exposes an OpenAI-compatible endpoint on localhost, translates request formats between providers, tracks quota, and automatically falls back across a subscription → cheap → free tier so work continues when one provider hits a limit. It also compresses tool-result content before sending it to the model to reduce input tokens. The audience is developers who use multiple providers or subscriptions and want a single routing layer with a dashboard.

## Key features

- RTK Token Saver: auto-detects tool outputs (git diff, git status, grep, find, ls, tree, logs) and applies compression before the request reaches the LLM; it runs before format translation and silently keeps the original text if a filter fails or grows the output.
- Three-tier fallback across subscription, cheap, and free providers, configured as named "combos" that switch automatically on quota exhaustion or errors.
- Format translation between OpenAI, Claude, Gemini, Cursor, Kiro, Vertex, Antigravity, Ollama, and OpenAI Responses formats.
- Multi-account support per provider with round-robin or priority routing, plus automatic OAuth token refresh.
- Real-time quota tracking, usage analytics, request logging/debug mode, and cloud sync of config across devices.
- Optional Headroom integration (an external `/v1/compress` proxy) and prompt-injection modes (Caveman, Ponytail) for reducing output tokens.
- Multiple deployment targets: localhost, VPS, Docker, and Cloudflare Workers.

## Tech stack

- Primary language: JavaScript; the web dashboard is a Next.js application (`next ^16.1.6`) with React `19.2.4`.
- Server/runtime: `express ^5.2.1`, `http-proxy-middleware ^3.0.5`, `undici ^7.19.2`, `jose ^6.1.3` for JWTs, `node-forge` and `selfsigned` for certificates, `socks-proxy-agent`.
- Storage: `better-sqlite3 ^12.6.2` kept as an optional dependency with `sql.js ^1.14.1` as a runtime fallback for systems without build tools.
- UI: `@monaco-editor/react`, `@xyflow/react`, `recharts`, `@dnd-kit/*`, `material-symbols`, `zustand`, Tailwind CSS v4.
- Distributed as a global npm CLI (`npm install -g 9router`) and via Docker/GHCR images; a separate `cli/` package builds the terminal UI and system tray.

## When to reach for it

- You use one or more AI coding CLIs and want a single local endpoint that routes across many providers.
- You want automatic failover when a subscription or free tier runs out of quota mid-task.
- You juggle multiple accounts or subscriptions and want round-robin load balancing and quota visibility.
- You want to cut the token cost of large tool outputs without changing your client.

## When *not* to reach for it

- You use a single provider and model steadily and do not need routing or fallback.
- You need a stable, versioned API — the app package is pre-1.0 (`0.5.40`) and carries a very large open-issue count.
- You are uncomfortable routing provider credentials and OAuth tokens through a local proxy layer.
- You are latency-sensitive, since the router adds a network hop plus format translation and compression on each request.

## Maturity signal

9Router gained a large audience quickly, reaching nearly 23,000 stars within roughly half a year of its creation, and it is under very active development with frequent pushes. At the same time it is pre-1.0 and carries over a thousand open issues, which points to rapid, still-stabilizing development rather than a settled release. Treat it as fast-moving and popular but early: expect breaking changes and churn, and pin versions if you depend on it.

## Alternatives

- LiteLLM — use it when you want a widely adopted, provider-agnostic proxy focused on a unified API and routing rather than a coding-tool dashboard.
- OpenRouter — reach for it when you prefer a hosted aggregation service over running a local router (9Router can itself route through it as one provider).
- Portkey — consider it when you want a gateway emphasizing observability, guardrails, and caching for production API traffic.

## Notes

The root npm package is private (`9router-app`), so the README treats running from source or Docker as the expected local development path, while the published `9router` npm package is the CLI. The dashboard's displayed "cost" is explicitly a tracking and savings estimate, not a bill — 9Router itself never charges. The README also documents that several free provider tiers (iFlow, Qwen Code, Gemini CLI) were discontinued in 2026 and that Kiro AI moved to a paid model, so the free-tier landscape it targets shifts frequently.

## Tags

javascript, nextjs, llm-gateway, proxy, ai-coding, cli, api-gateway, token-optimization, self-hosted, docker
