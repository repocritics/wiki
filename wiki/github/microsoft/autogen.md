# microsoft/autogen

> A Python (and .NET) framework for building multi-agent LLM applications — now in maintenance mode, superseded by Microsoft Agent Framework.

[GitHub repo](https://github.com/microsoft/autogen) ·
[Official website](https://microsoft.github.io/autogen/) ·
[License: MIT (code) / CC-BY-4.0 (docs)](https://github.com/microsoft/autogen/blob/main/LICENSE-CODE)

## Overview

AutoGen is a framework for orchestrating multiple LLM-backed agents that converse, call tools, execute code, and coordinate to complete tasks. It began as a Microsoft Research project in 2023 and became one of the earliest and most-cited multi-agent frameworks, popularizing patterns like two-agent chat, group chat with a manager, and agents that write and run their own code[^1]. As of this writing it has ~59.7k stars and ~9.0k forks.

The single most important fact about AutoGen today: **it is in maintenance mode.** The maintainers state it will receive no new features and is community-managed, and they direct new projects to Microsoft Agent Framework (MAF), described as the enterprise successor[^2]. This is not a deprecation of a dead library — AutoGen still works and is widely deployed — but it is a strong signal that the code you write against it is a terminal branch, not a growing one.

The second most important fact is that AutoGen underwent a full ground-up rewrite. The pre-2025 `v0.2` line (PyPI package `pyautogen`) and the current `v0.4+` line (packages `autogen-agentchat`, `autogen-core`, `autogen-ext`) are architecturally different frameworks that share a name[^3]. Most tutorials, blog posts, and Stack Overflow answers written before 2025 describe the old API and do not run on the new one. Anyone adopting or debugging AutoGen must first determine which version they are on.

## Getting Started

```bash
# Current (v0.4+) line — async, layered
pip install -U "autogen-agentchat" "autogen-ext[openai]"
```

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="Say 'Hello World!'"))
    await model_client.close()

asyncio.run(main())  # requires OPENAI_API_KEY
```

AutoGen requires Python 3.10 or later. The API is `async`-first; there is no synchronous convenience wrapper, so entry points must run inside an event loop.

## Architecture / How It Works

The v0.4 framework is deliberately layered, and understanding the layers is the key to using it:

- **Core (`autogen-core`)** — an event-driven, actor-model runtime for message passing between agents. Agents are addressed by identity, communicate via typed messages, and can run in a single process or a distributed runtime. Core has cross-language ambitions: it exists in both Python and .NET, with a gRPC gateway for distributed setups[^4].
- **AgentChat (`autogen-agentchat`)** — an opinionated higher-level API built on Core, aimed at rapid prototyping. This is where `AssistantAgent`, teams, and multi-agent orchestration patterns (round-robin group chat, selector-based routing, `AgentTool`) live. It is the closest analog to what v0.2 users knew.
- **Extensions (`autogen-ext`)** — first- and third-party integrations: model clients (OpenAI, Azure OpenAI, others), code executors, MCP tool workbenches, and more. Installed via extras like `autogen-ext[openai]`.

On top of the framework sit developer tools: **AutoGen Studio** (a no-code GUI for wiring agent teams), **AutoGen Bench / agbench** (agent evaluation harness), and **Magentic-One**, a reference generalist multi-agent team for web browsing, code execution, and file tasks.

The actor/event-driven core is a genuine departure from v0.2's synchronous, conversation-centric design. It buys concurrency and distribution but raises the conceptual floor: message types, subscriptions, and runtime lifecycle are now first-class concerns even for simple apps. Most users stay entirely within AgentChat and never touch Core directly.

## Production Notes

- **Maintenance mode is the headline risk.** No new features, community-managed issue response, and an explicit push toward MAF[^2]. For a greenfield production system, weigh whether to build on a framework whose maintainers are steering you elsewhere. The migration guide from AutoGen to MAF exists precisely because this transition is expected[^2].
- **The org fork confusion.** AutoGen's original creators (Chi Wang and Qingyun Wu) left Microsoft and continued a fork under the `ag2ai/ag2` project, also historically shipping the `pyautogen`/`autogen` PyPI names[^5]. This produced a period of real ambiguity about which "AutoGen" a given `pip install autogen` resolved to. Pin explicit package names (`autogen-agentchat`) and verify the source before assuming which project's code you are running.
- **v0.2 → v0.4 is a rewrite, not an upgrade.** There is a migration guide, but it is a porting exercise, not a version bump[^3]. Group chat semantics, agent construction, and the execution model all changed.
- **AutoGen Studio is explicitly not production software.** The maintainers label it a prototyping and demo tool with no built-in auth or hardening; do not expose it as an end-user app[^1].
- **Code execution is a live security surface.** Agents that write and run code (a core selling point) execute that code somewhere. Use the Docker-based executors rather than local execution for any untrusted input, and treat MCP server connections as trust boundaries — the docs warn that MCP servers can run commands in your environment[^1].
- **License is split.** Repository code is MIT; documentation and non-code content are CC-BY-4.0 (which is why GitHub's detected SPDX for the repo reads `CC-BY-4.0`). Do not assume the whole repo is CC-BY when vendoring code[^1].

## When to Use / When Not

**Use when:**
- You are maintaining or extending an existing AutoGen deployment and a rewrite to MAF is not yet justified.
- You want the actor-model, distributed multi-agent runtime specifically, and value the Core API's message-passing model.
- You need AutoGen Studio or Magentic-One as a research/prototyping starting point.

**Avoid when:**
- You are starting a new production project — the maintainers themselves point to Microsoft Agent Framework[^2].
- You want a long-lived, actively feature-developed framework; that is now MAF, LangGraph, or CrewAI, not AutoGen.
- You are following older tutorials and cannot confirm they target v0.4 — the version mismatch will waste more time than it saves.

## Alternatives

- microsoft/agent-framework — the maintainer-designated successor; choose it for any new multi-agent project intended for long-term support.
- ag2ai/ag2 — the original-creators' fork; use it if you specifically want continuity with the v0.2 lineage and its community direction.
- langchain-ai/langgraph — graph-based agent orchestration; use it when you want explicit state-machine control over agent flow rather than conversation abstractions.
- crewAIInc/crewAI — role/task-oriented crews; use it when a lighter, more prescriptive team model fits better than AutoGen's layers.
- openai/openai-agents-python — vendor-native agent SDK; use it when you are OpenAI-only and want the smallest surface area.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 / initial | 2023-08 | Repository created; Microsoft Research multi-agent conversation framework[^1]. |
| 0.2 (`pyautogen`) | 2023–2024 | Widely adopted synchronous, conversation-centric API; basis of most early tutorials[^3]. |
| 0.4 rewrite | 2025-01 | Ground-up redesign: async, event-driven actor Core + AgentChat + Extensions layers[^3][^4]. |
| Maintenance mode | 2025+ | Declared community-managed; new work directed to Microsoft Agent Framework[^2]. |

*Exact point-release dates vary across the three PyPI packages; consult the GitHub releases page for a given package version.*

## References

[^1]: AutoGen README and documentation. https://microsoft.github.io/autogen/
[^2]: Maintenance-mode notice and migration guidance, AutoGen README; Microsoft Agent Framework. https://github.com/microsoft/agent-framework and https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/
[^3]: AutoGen v0.2 → v0.4 migration guide. https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/migration-guide.html
[^4]: Core API package (message passing, distributed runtime, .NET + Python). https://github.com/microsoft/autogen/tree/main/python/packages/autogen-core
[^5]: AG2 (community fork by original creators). https://github.com/ag2ai/ag2

## Tags

python, dotnet, multi-agent, llm, agent-framework, ai-agents, orchestration, microsoft, maintenance-mode, mcp, code-execution
