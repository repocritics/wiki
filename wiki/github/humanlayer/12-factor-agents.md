# humanlayer/12-factor-agents

A written set of twelve engineering principles for building LLM-powered software reliable enough to ship to production customers.

## What it is

12-factor agents is a methodology guide, in the spirit of the 12 Factor Apps manifesto, that catalogs the design patterns its author found while building agents and talking to founders shipping them. Rather than proposing another framework, it argues that good agents are "mostly just software" and that the durable value comes from small, modular concepts a competent engineer can apply to an existing product. The audience is software engineers building customer-facing LLM features who have hit the quality ceiling of plug-and-play frameworks. The core content is a series of Markdown chapters, one per factor, plus supporting scaffolding and workshop material.

## Key features

- Twelve documented factors, each in its own chapter under `content/`, covering topics such as natural-language-to-tool-calls, owning your prompts, owning your context window, treating tools as structured outputs, unifying execution and business state, launch/pause/resume APIs, contacting humans via tool calls, owning control flow, compacting errors, small focused agents, triggering from anywhere, and the stateless-reducer pattern.
- A conceptual model of the agent loop as: LLM chooses the next step, deterministic code executes it, the result is appended to context, repeat until done.
- A `create-12-factor-agent` scaffolding package (invocable via `npx`/`uvx`) with a template project that includes BAML sources and TypeScript agent, CLI, and server files.
- A `walkthroughgen` package for generating step-by-step code walkthroughs, with TypeScript examples.
- Hands-on workshop material organized into progressive sections (hello-world, CLI-and-agent, calculator-tools, tool-loop).
- An appendix factor ("pre-fetch all the context you might need") and a curated list of related resources.

## Tech stack

- Primary language: TypeScript; the author notes the patterns are language-agnostic and work in Python or other languages.
- BAML (`baml_src/` files) used in the scaffolding template and workshops for defining agents, clients, and tools.
- Node/npm packages (`package.json`) for the template project and the `walkthroughgen` tool, which uses Jest for tests.
- A small Python helper under `hack/contributors_markdown` managed with `uv` and `pyproject.toml`.

## When to reach for it

- You are adding agentic behavior to an existing production application and want proven patterns instead of a framework rewrite.
- You have reached the ~70–80% quality bar with a framework and need the finer control to push past it.
- You want a shared vocabulary and reference for reviewing agent designs on a team.
- You want a minimal scaffold to start a new agent project along these principles.

## When *not* to reach for it

- You want a batteries-included runtime that manages orchestration, memory, and tools for you — this is guidance, not a library to depend on.
- You need drop-in infrastructure code; most of the repository is prose and examples rather than a maintained package.
- You are looking for coverage of MCP specifically — the author explicitly sets that topic aside.

## Maturity signal

With over 24,000 stars and a large fork count, this is a widely referenced piece of writing that clearly resonated with practitioners. It is not archived, but the last push was several months before this listing, which fits its nature: a conceptual guide reaches a stable, largely finished state rather than requiring continuous releases. Treat it as a settled, well-regarded reference whose value is the ideas, not an actively iterating codebase.

## Alternatives

- LangGraph — reach for it when you want an actual orchestration framework with graph-based control flow rather than a set of principles to implement yourself.
- CrewAI — use it when you want a higher-level, plug-and-play multi-agent abstraction and can accept less low-level control.
- smolagents — use it when you want a deliberately minimal agent library to build on directly instead of hand-rolling the loop.
- BAML — use it when you want the structured-output and prompt-as-function tooling that this guide's examples themselves rely on.

## Notes

The project uses split licensing, which is why its SPDX field is unresolved: prose and images are under CC BY-SA 4.0 while code is under Apache 2.0. The central thesis is contrarian for its space — it argues against going all-in on agent frameworks and observes that many products marketed as "AI agents" are mostly deterministic code with LLM steps inserted at key points.

## Tags

typescript, llm, ai-agents, prompt-engineering, methodology, guide, tool-calling, context-engineering, orchestration, baml
