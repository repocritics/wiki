# crewAIInc/crewAI

> A Python framework for orchestrating role-playing autonomous agents (Crews) and event-driven workflows (Flows), independent of LangChain.

[GitHub repo](https://github.com/crewAIInc/crewAI) ·
[Official website](https://crewai.com) ·
[License: MIT](https://github.com/crewAIInc/crewAI/blob/main/LICENSE)

## Overview

CrewAI is a multi-agent orchestration framework created by João Moura in late 2023[^1]. Its core abstraction is a *crew*: a set of `Agent`s, each given a natural-language `role`, `goal`, and `backstory`, that collaborate on `Task`s via a `Process` (sequential or hierarchical). A second primitive, *Flows*, was added later to provide explicit event-driven control — decorators like `@start`, `@listen`, and `@router` wire deterministic Python steps and single LLM calls around the more autonomous crews.

The framework's defining decision is that it was rewritten to drop its original LangChain dependency and stand as an independent core built on LiteLLM for model access[^2]. That decoupling is the source of both its appeal (a smaller dependency surface, prompts and execution paths you can actually read) and its risk (it reimplements agent, tool, and memory machinery that larger ecosystems maintain separately). CrewAI markets heavily on developer volume — over 100,000 people certified through its community courses[^3] — and pairs the open-source library with a commercial control plane (AMP Suite / Crew Control Plane) for hosted deployment and observability.

The central tension is autonomy versus determinism. Crews optimize for emergent, self-directed collaboration; Flows exist because pure autonomy is hard to ship to production. Most real CrewAI systems end up as Flows that call Crews at the points where open-ended reasoning actually helps.

## Getting Started

Requires Python `>=3.10 <3.14`. The project standardizes on [uv](https://docs.astral.sh/uv/) for dependency management.

```shell
uv pip install crewai            # core
uv pip install 'crewai[tools]'   # adds crewai_tools integrations
crewai create crew my_project    # scaffolds config/agents.yaml, tasks.yaml, crew.py, main.py
```

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="Senior Data Researcher",
    goal="Find cutting-edge developments in {topic}",
    backstory="A seasoned researcher known for clear, relevant synthesis.",
    verbose=True,
)

task = Task(
    description="Research {topic} and return 10 key bullet points.",
    expected_output="A markdown list of 10 bullets.",
    agent=researcher,
)

crew = Crew(agents=[researcher], tasks=[task], process=Process.sequential)
result = crew.kickoff(inputs={"topic": "AI agents"})
```

The scaffolded project keeps agent and task definitions in YAML and wires them with the `@CrewBase`, `@agent`, `@task`, `@crew` decorators, so prompt content lives outside code. Run with `crewai run`.

## Architecture / How It Works

An `Agent` is fundamentally a prompt template plus a loop. Role/goal/backstory are interpolated into a system prompt; the agent then runs a ReAct-style tool-use loop against an LLM until the task's `expected_output` is satisfied. Model access goes through **LiteLLM**, so any provider LiteLLM supports (OpenAI, Anthropic, Google, Ollama, Bedrock, etc.) is reachable by setting a model string and the matching API-key env var[^2].

`Process` selects the coordination strategy. `Process.sequential` runs tasks in order, threading each task's output into the next as context. `Process.hierarchical` injects an auto-generated *manager* agent that plans, delegates to workers, and validates results — more capable but less predictable, and it spends extra LLM calls on the coordination layer itself.

**Flows** are the newer, more deterministic half. A `Flow` is a Python class whose methods are annotated to react to events; state is carried in a typed (often Pydantic) model between steps, and `or_` / `and_` combinators build branching conditions. Flows are where you put business logic, human-in-the-loop gates, and structured I/O; Crews are what you invoke from inside a Flow when a step genuinely needs open-ended agent reasoning.

Supporting subsystems: **tools** (`crewai_tools`, a separate package of prebuilt integrations plus a `BaseTool` interface for custom ones), **memory** (short-term, long-term, and entity memory backed by a vector store, typically ChromaDB, using embeddings), **knowledge** (document sources chunked and retrieved at run time), and **guardrails** for validating task output. Structured output is available via `output_pydantic` / `output_json`. MCP and A2A support connect crews to external tool servers and other agents.

## Production Notes

**Telemetry is on by default.** CrewAI collects anonymous usage telemetry unless you opt out (set `CREWAI_DISABLE_TELEMETRY=true`, or historically `OTEL_SDK_DISABLED=true`)[^4]. Audit this before deploying in regulated or air-gapped environments; the README documents what is and isn't collected.

**Cost and latency are emergent, not bounded.** Autonomous crews decide how many LLM calls to make. A hierarchical crew multiplies this — the manager agent reasons on every delegation. There is no built-in hard cap on total tokens for a `kickoff()`; teams that ship CrewAI to production typically wrap it in Flows with explicit step limits, set `max_iter` / `max_rpm` per agent, and add their own budget guards. Non-determinism makes reproducing a bad run hard; enable `verbose=True` and capture traces early.

**The LangChain decoupling cuts both ways.** Because the core is independent, dependency conflicts are fewer, but you also inherit CrewAI's own reimplementations of memory, tool routing, and prompt assembly. Behavior across minor versions has shifted as that core matured.

**Version churn.** CrewAI moved fast through its 0.x line; APIs, the CLI, and defaults changed between minor releases, and older projects reference a Poetry-based flow that later switched to uv (`crewai update` exists to migrate). Pin the version, read release notes before upgrading, and expect prompt-level behavior to drift even when your code is unchanged.

**Common install snags:** `tiktoken` needs a Rust toolchain to build from source (`crewai[embeddings]` or a prebuilt wheel avoids it); embedding-backed memory pulls in ChromaDB and its own dependency weight.

## When to Use / When Not

**Use when:**
- You want role-based multi-agent collaboration with minimal boilerplate and YAML-configured prompts.
- Your problem decomposes into specialist agents (researcher, writer, reviewer) handing work down a pipeline.
- You want a lighter dependency footprint than LangChain-based stacks, with a clear commercial hosting path if needed.
- You can combine Flows (control) with Crews (autonomy) rather than relying on autonomy alone.

**Avoid when:**
- You need tight, auditable control over every state transition — a graph/state-machine framework fits better.
- Bounded, predictable cost per request is a hard requirement and you don't want to build the guardrails yourself.
- Your task is a single LLM call or a fixed deterministic pipeline; the agent abstraction is overhead.
- You require long-term API stability; the framework still iterates quickly.

## Alternatives

- langchain-ai/langgraph — graph/state-machine orchestration; use when you need explicit, auditable control over transitions rather than agent autonomy.
- microsoft/autogen — conversational multi-agent framework; use when research-style agent-to-agent dialogue is the point.
- openai/openai-agents-python — lightweight provider-native agents SDK; use when you're OpenAI-centric and want minimal abstraction.
- pydantic/pydantic-ai — type-first, single-agent-leaning framework; use when structured output and static typing matter more than multi-agent choreography.
- langchain-ai/langchain — broad LLM toolkit; use when you want the largest integration ecosystem and don't mind the dependency weight.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2023-11 | Initial release; role-based agent crews, originally built on LangChain[^1]. |
| 0.x | 2024 | Core rewritten to remove the LangChain dependency; LiteLLM adopted for model access[^2]. |
| 0.7x+ | 2024-10 | Flows introduced — event-driven `@start`/`@listen`/`@router` control layer. |
| 0.x | 2025 | uv-based tooling, memory/knowledge/guardrails maturation, MCP + A2A support. |
| — | 2026 | AMP Suite / Crew Control Plane positioned as commercial layer; repo actively maintained (last push 2026-07). |

## References

[^1]: crewAI repository and history, created 2023-10-27. https://github.com/crewAIInc/crewAI
[^2]: CrewAI docs, "Connect to any LLM" (LiteLLM-based model access) and the framework's independence from LangChain. https://docs.crewai.com/concepts/llms
[^3]: CrewAI README, community certification figure. https://learn.crewai.com
[^4]: CrewAI docs, "Telemetry" — anonymous data collection and opt-out. https://docs.crewai.com/telemetry

## Tags

python, ai-agents, multi-agent, agent-orchestration, llm, workflow-engine, litellm, framework, autonomous-agents, agentic
