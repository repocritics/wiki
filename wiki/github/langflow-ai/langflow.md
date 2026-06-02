# langflow-ai/langflow

A visual workflow builder for LLM-powered agents and pipelines — drag-and-drop node graphs that compile to runnable Python.

## What it is

A Python + React-Flow application that lets engineers compose LLM workflows (prompt chains, RAG pipelines, agent loops, tool use) as visual node graphs. Each node represents a component (LLM call, retriever, parser, tool, custom function), and edges express the data flow between them. The visual graph compiles to executable Python that can be exported, hosted, or wrapped into a REST endpoint. Targets the gap between "code-first" (LangChain) and "fully-managed-SaaS" (Vercel AI / Make.com).

## Key features

- React-Flow-based visual editor for building LLM workflows.
- Component library for LLM calls, retrievers, parsers, agents, memory, tools, and custom Python blocks.
- Export workflows as REST endpoints / Python scripts.
- Multi-agent orchestration — agents as first-class nodes in the graph.
- Self-host (Docker, Python install) or cloud at langflow.org.
- MIT-licensed.

## Tech stack

- Python primary on the runtime side.
- React + React-Flow on the frontend; TypeScript.
- LangChain (and related ecosystem packages) as the underlying composition substrate.

## When to reach for it

- You're designing LLM workflows with a team that includes non-coders and want a shared visual artifact.
- You're prototyping agent / RAG flows and want fast iteration without writing boilerplate.
- You're operating workflows in production and want a graph-first audit surface.

## When *not* to reach for it

- You want lightweight, code-only LLM composition — use LangChain or DSPy directly.
- You need extreme performance — Langflow's overhead is fine for prototyping but adds latency vs. hand-rolled pipelines.
- You want vendor-lock-free portability — workflows depend on Langflow's runtime to execute the graph.

## Maturity signal

149k stars, 9k forks, MIT, last push the day this page was generated. 3-year-old project under the langflow-ai organization (formerly part of `logspace-ai`). The 931 open-issues count tracks LangChain-ecosystem churn — Langflow follows upstream API changes which surface as integration bugs. Active development cadence; Langchain's commercial parent has acquired interest in the project.

## Alternatives

- LangChain / LangGraph (code-first) — use when you want library-style composition.
- n8n + AI nodes — use when you want general workflow automation with LLM as one of many integrations.
- Flowise — direct competitor with similar visual-editor approach.
- AnythingLLM, Dify — use when you want hosted/turnkey RAG + chat rather than visual workflow building.

## Notes

The Langflow ↔ LangChain coupling is the strongest implementation choice — when LangChain APIs change, Langflow follows. Anyone choosing between Langflow and Flowise should weight which underlying ecosystem they want to depend on. MIT license keeps the code portable, but the runtime dependency on Langflow's graph-execution engine is the real lock-in.

## Tags

artificial-intelligence, large-language-model, agent, workflow-automation, python, react, low-code, langchain, retrieval-augmented-generation, framework
