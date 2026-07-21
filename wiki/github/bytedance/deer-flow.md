# bytedance/deer-flow

An open-source super-agent harness that orchestrates sub-agents, memory, and sandboxes to run long-horizon tasks that span from minutes to hours.

## What it is

DeerFlow (Deep Exploration and Efficient Research Flow) is a self-hostable harness for building and running agents that research, write code, and create content over extended sessions. It is aimed at developers who want to operate a persistent agent runtime with tools, skills, sub-agents, and isolated execution rather than a single one-shot chat call. Version 2.0 is a ground-up rewrite that shares no code with the original 1.x Deep Research framework, which is now maintained on a separate branch.

## Key features

- Sub-agent orchestration with a lead agent that delegates to subordinate agents, plus configurable per-run caps such as `subagents.max_total_per_run`.
- Multiple sandbox execution modes: directly on the host, in isolated Docker containers, or in Kubernetes pods via a provisioner service.
- Extensible skills and configurable MCP servers, including HTTP/SSE OAuth token flows and per-tool call timeouts for stdio servers.
- Inbound task channels from messaging apps: Telegram, Slack, Feishu/Lark, WeChat, WeCom, and DingTalk, none of which require a public IP.
- Long-term memory, manual context compaction, session goals, and context engineering features for managing long-running state.
- Broad model provider support through LangChain, including OpenAI, OpenRouter and other OpenAI-compatible gateways, vLLM, and CLI-backed providers (Codex CLI, Claude Code OAuth).

## Tech stack

- Python 3.12+ backend.
- Node.js 22+ frontend, built with pnpm.
- LangChain and LangGraph for agent orchestration; Gateway exposes LangGraph-compatible `/api/langgraph/*` paths.
- Docker and Docker Compose for the recommended deployment; Kubernetes supported for sandbox execution.
- nginx as the bundled unified gateway endpoint.
- SQLite or Postgres as the shared backend for the LangGraph checkpointer, LangGraph Store, and application data.
- uv for Python dependency management; `make` targets drive setup, dev, and deployment.
- Optional tracing via LangSmith, Langfuse, and Monocle; optional Redis stream bridge for multi-worker SSE delivery.
- MIT licensed.

## When to reach for it

- You want a self-hosted agent runtime for tasks that run for minutes to hours rather than short single-turn requests.
- You need sub-agents, tool use, skills, and sandboxed code execution wired together out of the box.
- You want to drive an agent from chat platforms like Telegram, Slack, or Feishu without exposing a public callback URL.
- You want to mix model providers (hosted APIs, local vLLM, or CLI-backed Codex/Claude Code) behind one configuration.
- You are running a persistent shared server on Linux with Docker and can allocate meaningful CPU and memory headroom.

## When *not* to reach for it

- You need a lightweight library to embed in an existing app; this is a full multi-service system (backend, frontend, nginx, database, sandbox) with sizing guidance starting at 4 vCPU / 8 GB RAM and recommending more for server use.
- You are on macOS or Windows for production; the README treats those as development or evaluation environments and recommends Linux plus Docker for persistent servers.
- You want the original Deep Research pipeline; that code lives only on the `1.x` branch and shares nothing with 2.0.
- You need horizontal scaling of the Gateway; production defaults to a single Gateway worker because it owns active run tasks in process, and raising worker count requires sticky routing or explicit ownership work.
- Improper deployment can introduce security risks, so it is not a drop-in for untrusted or hardened multi-tenant environments without careful configuration.

## Maturity signal

DeerFlow is actively developed and heavily used. It was created in May 2025, was pushed to as recently as mid-2026, is not archived, and carries a permissive MIT license. The star count in the tens of thousands and thousands of forks indicate strong adoption, and the near-1,000 open issues are consistent with a large, fast-moving project under active maintenance rather than neglect. The explicit 2.0 rewrite and migration of active development off the 1.x branch signal an evolving codebase where APIs and internals may still shift.

## Alternatives

- LangGraph — reach for it directly when you want to build and control your own agent graph as a library rather than adopt a full harness; DeerFlow is built on top of it.
- OpenHands — use it when your primary goal is an autonomous software-engineering agent working in a repository, rather than a general research-and-create harness.
- CrewAI — use it when you want a lighter multi-agent orchestration framework to embed in your own Python application without running a full server stack.

## Notes

- The setup path is agent-friendly: the README includes a one-sentence prompt you can hand to Claude Code, Codex, Cursor, or another coding agent to clone and bootstrap the project, and `make support-bundle` generates AI-fileable issue drafts with redacted diagnostics.
- Sub-agent identity carries into chat channels: setting `assistant_id` to a custom agent name still routes through `lead_agent` but injects the name so the custom agent's SOUL/config takes effect for IM messages.
- ByteDance recommends specific models (Doubao-Seed-2.0-Code, DeepSeek v3.2, Kimi 2.5) via its Volcengine coding plan, but the model layer is provider-agnostic through LangChain.

## Tags

python, typescript, llm, ai-agents, multi-agent, agent-framework, langchain, langgraph, sandboxing, rag, self-hosted, harness
