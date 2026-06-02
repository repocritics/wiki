# langgenius/dify

A production-ready platform for agentic workflow development — visual + code-friendly LLM pipelines, RAG, agents, and tool orchestration with both no-code UI and full programmatic API.

## What it is

A TypeScript + Python platform that combines a visual workflow builder, a managed RAG pipeline, an agent framework, and an LLM-ops surface into one self-hostable application. The pitch is "production-ready" — the UI handles prototype-grade composition, while the runtime layer adds auth, observability, multi-tenant support, and API key management for serving workflows as endpoints. Hosted at dify.ai or self-hosted via Docker.

## Key features

- Visual workflow builder (Next.js + TypeScript) for composing LLM pipelines, agents, and RAG flows.
- Built-in RAG pipeline with document ingestion, chunking, embedding, retrieval.
- Agent framework with tool-calling, multi-step reasoning, and MCP integration.
- Multi-LLM-provider support (OpenAI, Anthropic Claude, Google Gemini, locally-hosted models).
- Production-ops surface: API keys, rate limits, multi-tenant workspaces, observability hooks.
- Self-hostable (Docker, Kubernetes) or hosted at dify.ai.
- App-template ecosystem — pre-built workflows for common use cases (chatbots, RAG, code review, etc.).

## Tech stack

- TypeScript primary on the frontend / API gateway.
- Python on the LLM-orchestration backend.
- Next.js for the visual builder.
- Docker-first deployment for self-host.

## When to reach for it

- You want production-ready LLM-app infrastructure without rolling your own auth + ops layer.
- Your team mixes engineers and non-engineers and needs a visual surface for collaboration.
- You're standing up internal LLM workflows that need multi-tenant access controls.

## When *not* to reach for it

- You're prototyping code-first LLM apps — LangChain, DSPy, or the OpenAI/Anthropic SDK directly are lighter.
- You're allergic to non-OSI licenses — SPDX `NOASSERTION` means specific commercial-use restrictions; verify LICENSE.
- You want a minimal, embedded library — Dify is a full platform with its own runtime.

## Maturity signal

143k stars, 22k forks, last push the day this page was generated. 3-year-old project with active commercial backing (LangGenius). The 815 open-issues count reflects multi-platform / multi-provider surface area — feature requests outweigh defect reports. Push cadence is rapid; the team ships new features and integrations on a near-weekly basis.

## Alternatives

- LangFlow, Flowise — direct competitors in the visual-LLM-workflow space.
- `n8n-io/n8n` — broader workflow automation with LLM nodes as one of many integration types.
- LangChain / LangGraph + a custom auth layer — use when you want code-first with full control over the ops surface.
- AnythingLLM, OpenWebUI — use when you only need RAG + chat rather than full workflow building.

## Notes

The "production-ready" framing is more accurate than the typical "demo-grade" LLM-platform pitch — the multi-tenant, API-key-managed ops surface is the differentiator. License is `NOASSERTION` — common for commercial-OSS hybrids; verify the LICENSE file before SaaS-hosting Dify as a competing service. MCP integration is recent and signals the project's alignment with the emerging multi-agent ecosystem.

## Tags

artificial-intelligence, large-language-model, agent, retrieval-augmented-generation, workflow-automation, low-code, typescript, python, nextjs, model-context-protocol, self-hosted, platform
