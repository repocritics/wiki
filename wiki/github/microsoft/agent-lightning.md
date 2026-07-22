# microsoft/agent-lightning

A Microsoft framework for training and optimizing AI agents — via reinforcement learning, automatic prompt optimization, or fine-tuning — with almost no changes to existing agent code.

## What it is

Agent Lightning is a trainer that turns an existing AI agent into an optimizable system with, in its own words, almost zero code change. It works with agents built on any framework (LangChain, OpenAI Agents SDK, AutoGen, CrewAI, Microsoft Agent Framework) or none at all, capturing prompts, tool calls, and rewards as structured spans that flow into a central store, then applying a chosen algorithm — reinforcement learning, automatic prompt optimization, or supervised fine-tuning — that posts back improved prompts or policy weights. It is for researchers and ML engineers who want to improve agents without rewriting them.

## Key features

- Instrument an existing agent with a lightweight emitter (`agl.emit_xxx()`) or an automatic tracer; events become structured spans.
- A central **LightningStore** keeps tasks, resources, and traces in sync; store backends include in-memory, SQLite, and MongoDB.
- A **Trainer** streams datasets to runners, ferries resources between the store and the algorithm, and updates the inference engine when improvements land.
- Algorithms span **reinforcement learning** (VERL integration), **automatic prompt optimization** (APO), and **supervised fine-tuning**, and you can selectively optimize one or more agents in a multi-agent system.
- Framework-agnostic adapters for LangChain, OpenAI Agents, AutoGen, CrewAI, and Anthropic, plus a plain Python/OpenAI path.
- Ships with a dashboard, a CLI (`agl`), and examples (SQL, RAG, ChartQA, and more).

## Tech stack

- Python >=3.10; packaged as `agentlightning` (version 0.3.1), built with hatchling.
- Core dependencies: fastapi, uvicorn, gunicorn, flask, aiohttp, pydantic >=2.11, litellm[proxy] >=1.74, openai, agentops >=0.4.13, and OpenTelemetry API/SDK/OTLP exporter >=1.35.
- Training-heavy extras (restricted to a Linux environment via uv): torch >=2.8 (or a pinned legacy 2.7), transformers, vllm (>=0.10.2 with specific exclusions), flash-attn, and VERL (>=0.6 stable / 0.5 legacy), plus Unsloth/TRL and Tinker integrations.
- Optional MongoDB store backend via the `mongo` extra (pymongo).
- MIT license.

## When to reach for it

- You have an existing agent on any framework and want to optimize it with RL, prompt optimization, or fine-tuning without rewriting it.
- You need to selectively train specific agents inside a multi-agent system.
- You are doing agent-RL research and want trace collection, a store, and VERL/vLLM training wired together.

## When *not* to reach for it

- You just want to build or orchestrate agents — this is a trainer, not an agent framework; use LangChain, AutoGen, or CrewAI directly for that.
- You cannot run a Linux GPU stack — the training path pins the environment to Linux and pulls CUDA torch, vLLM, and flash-attn.
- You want a low-churn dependency surface — the pyproject is dense with version pins, exclusions, and conflict rules that track a fast-moving RL and vLLM stack.
- You need only inference or serving, with no training loop.

## Maturity signal

Created in June 2025, Agent Lightning has roughly 17,400 stars, recent activity (last push July 2026), an MIT license, and is not archived. It is backed by Microsoft Research, with an arXiv paper (2508.03680) and a stated Responsible AI certification. The version (0.3.1) signals an early stage, but the extensive CI matrix, worked examples, and named community projects indicate active, serious maintenance. Read it as actively maintained and research-grade, but pre-1.0 with a moving dependency target.

## Alternatives

- **VERL** — the reinforcement-learning training library Agent Lightning integrates; use VERL directly when you want low-level RL training without the agent-tracing layer.
- **TRL / Unsloth / Tinker** — named as integrations and examples; reach for them directly when you want their fine-tuning path without Agent Lightning's orchestration.
- **LangChain / AutoGen / CrewAI** — use these to build and run agents; Agent Lightning sits on top to optimize agents built with them, so it complements rather than replaces them.

## Notes

- The central claim is optimizing an agent with almost zero code change by tracing it rather than rewriting it.
- It is a trainer, not an agent framework; it wraps whatever framework you already use.
- The uv configuration restricts training environments to Linux and encodes many version conflicts (torch-stable versus legacy, specific vLLM pins), which shows the RL/vLLM stack is fragile to version drift.
- The store has swappable backends: in-memory, SQLite, and MongoDB.

## Tags

python, machine-learning, reinforcement-learning, llm, ai-agents, mlops, fine-tuning, framework
