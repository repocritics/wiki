# lobehub/lobehub

A "Chief Agent Operator" platform — pitches itself as scheduling, hiring, and reporting on an entire AI team of agents running 24/7.

## What it is

A TypeScript platform for orchestrating multiple AI agents across providers (OpenAI, Claude, Gemini, DeepSeek). Adds a management layer ("CAO" — Chief Agent Operator) on top of conventional chat / agent UIs. Tightly integrated with the LobeChat ecosystem (separate `lobehub/lobe-chat` repo). License `NOASSERTION` — verify before commercial reuse.

## Key features

- Multi-agent orchestration with role definitions, scheduling, reporting.
- Multi-provider LLM support: OpenAI, Claude, Gemini, DeepSeek.
- MCP integration.
- Skills + knowledge-base system per agent.
- Active push cadence; tight coupling with the broader LobeHub product family.

## Tech stack

- TypeScript primary.

## When to reach for it

- You're managing multiple AI agents and want a coordination layer rather than running them ad-hoc.
- You're already in the LobeChat ecosystem.

## When *not* to reach for it

- You want vendor-supported orchestration — pick first-party offerings.
- You're allergic to non-OSI licenses — verify LICENSE.

## Maturity signal

78k stars, 15k forks, last push very recent. License `NOASSERTION` is the recurring caveat.

## Alternatives

- LangGraph — code-first multi-agent orchestration.
- AutoGen, CrewAI — alternative multi-agent frameworks.
- `langflow-ai/langflow`, `langgenius/dify` — visual workflow alternatives.

## Tags

artificial-intelligence, large-language-model, agent, typescript, multi-agent, claude, openai, gemini, model-context-protocol
