# browser-use/browser-use

A Python library and CLI that lets an LLM-driven agent operate a real web browser to complete tasks described in natural language.

## What it is

Browser Use gives an AI agent control of a web browser so it can open pages, click buttons, type, and fill in forms the way a person would. You describe a task in plain language and the agent carries it out, using an LLM of your choice. It targets developers who want to automate web workflows from code, and users who want an existing coding agent (Claude Code, Codex, Cursor, Hermes, OpenClaw) to complete browser tasks for them.

## Key features

- Natural-language browser automation: an `Agent` runs an async loop to fill forms, extract structured data, and QA-test websites from a task description.
- Provider-agnostic LLM support with bundled SDKs for OpenAI, Anthropic, Google, Groq, Ollama, DeepSeek, Mistral, Cerebras, AWS Bedrock, Azure, OpenRouter, LiteLLM, and OCI.
- `ChatBrowserUse` accepts provider-prefixed model ids so a single `BROWSER_USE_API_KEY` can reach multiple providers, alongside Browser Use's own optimized `bu-*` models and an open-source preview model.
- CLI distributed as an installable "skill" (`browser-use skill install`) that lets other agents drive the browser, with entry points `browser-use`, `browseruse`, `bu`, and `browser`.
- Custom tools: register your own actions with a `Tools().action` decorator to extend agent capabilities.
- Built-in integrations including MCP (`mcp` dependency) and Gmail, plus optional pairing with Browser Use Cloud browsers for stealth, proxy rotation, and scaling.

## Tech stack

- Python, requires `>=3.11,<4.0`; packaged with the hatchling build backend.
- Pydantic 2.12.5 for data models; async stack built on aiohttp, anyio, and httpx.
- Browser control via `cdp-use` 1.4.5 (Chrome DevTools Protocol); GitHub topics also list Playwright.
- LLM client libraries: openai 2.16.0, anthropic 0.76.0, google-genai 1.65.0, groq 1.0.0, ollama 0.6.1, plus `mcp` 1.26.0.
- CLI/terminal tooling: click 8.3.1, rich 14.3.1, InquirerPy 0.3.4.
- First-party helpers `browser-use-sdk` 3.4.2 and `browser-harness` 0.1.6, with an optional native `browser-use-core` extra for macOS/Linux/Windows.
- Optional extras: `aws` (boto3), `oci`, and `video` (imageio, numpy).

## When to reach for it

- Building repeatable web automation in code, such as scraping, monitoring, or QA runs on a schedule or in parallel.
- Embedding a browser agent inside your own product.
- Giving an existing coding agent the ability to complete one-off browser tasks through the CLI.
- Form filling, structured-data extraction to formats like CSV, and automated QA testing of a website.
- Cases needing custom tools, custom system prompts, structured output, or fine-grained browser control.

## When *not* to reach for it

- High-volume production with many parallel Chrome instances: the README notes Chrome is memory-heavy and recommends the hosted Cloud API for scalable infrastructure and memory management.
- Workloads that must defeat CAPTCHAs or bot detection, which the README says require better fingerprinting and proxies via Browser Use Cloud rather than the open-source agent alone.
- Non-Python stacks, since the library is Python-only.
- Deterministic, low-cost scraping where an LLM agent is unnecessary, given that every run requires choosing an LLM provider.

## Maturity signal

Actively maintained and widely adopted. The repository has roughly 106k stars, was created in late 2024, and had its most recent push one day before these facts were captured, with no archive flag and an MIT license. The pre-1.0 version number (0.13.6) suggests the public API may still change between releases despite the large user base, so pin versions for production use.

## Alternatives

- Playwright — use it instead when you want scripted, deterministic browser automation without an LLM agent in the loop.
- Stagehand — use it instead when you prefer a TypeScript-first AI browser automation framework (referenced as an optional example dependency).
- Browserbase — use it instead when you primarily need hosted, managed browser infrastructure to run sessions at scale (referenced as an optional example dependency).

## Notes

- A single `BROWSER_USE_API_KEY` can route to OpenAI, Anthropic, and Google models through `ChatBrowserUse`, avoiding separate per-provider keys.
- Beyond the hosted `bu-*` models there is an open-source preview model (`browser-use/bu-30b-a3b-preview`) that still receives the default agent system prompt automatically.
- The project reports a benchmark of 100 real-world tasks and claims the top position on the Odysseys leaderboard for long-horizon web tasks; the benchmark is published as a separate open-source repository.

## Tags

python, browser-automation, ai-agents, llm, cli, library, playwright, mcp, web-scraping, automation
