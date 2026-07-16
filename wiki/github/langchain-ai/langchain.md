# langchain-ai/langchain

> A Python framework for building LLM applications and agents by composing models, tools, and integrations behind a common interface.

[GitHub repo](https://github.com/langchain-ai/langchain) ·
[Official website](https://docs.langchain.com/langchain/) ·
[License: MIT](https://github.com/langchain-ai/langchain/blob/master/LICENSE)

## Overview

LangChain, created by Harrison Chase in October 2022[^1], is the most widely adopted framework for building applications on top of large language models. Its core promise is *model interoperability*: a standard interface for chat models, embeddings, vector stores, retrievers, and tools, so that swapping OpenAI for Anthropic (or a local model) is a one-line change rather than a rewrite. The project sits at the center of a larger ecosystem — LangGraph (agent orchestration), LangSmith (commercial observability/eval), and LangChain.js (the TypeScript port) — that the same company builds and monetizes.

The framework's defining tension is *abstraction vs. transparency*. LangChain's convenience comes from wrapping provider APIs, prompt templates, retrieval, and control flow behind uniform classes. When those abstractions fit your use case they save real work; when they don't, you are debugging through several layers of indirection to reach the actual HTTP call. The project has been criticized for exactly this, and its own answer has been to keep pushing complexity *down* into thin, swappable packages (`langchain-core`, per-provider partner packages) and control flow *out* into LangGraph.

The repository has changed identity over its life. Early LangChain was a grab-bag of "chains" and prompt utilities; the current line (v1) rebrands as "the agent engineering platform"[^2] and centers on `create_agent`, standardized message/content blocks, and a much thinner core that delegates real orchestration to LangGraph. This churn is the single most important thing to understand before adopting it: the API you learn this year is not guaranteed to be the API next year.

## Getting Started

```bash
pip install langchain
# or, as the README recommends:
uv add langchain
```

A minimal model call using the provider-agnostic initializer:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("anthropic:claude-sonnet-4-5")
print(model.invoke("Explain retrieval-augmented generation in one sentence.").content)
```

A tool-calling agent (v1 API, built on LangGraph under the hood):

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """Return the current weather for a city."""
    return f"It's sunny in {city}."

agent = create_agent("openai:gpt-5.5", tools=[get_weather])
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in Seoul?"}]}
)
print(result["messages"][-1].content)
```

## Architecture / How It Works

LangChain is not one package but a family, split deliberately to isolate dependency risk:

- **`langchain-core`** — the base abstractions: `BaseChatModel`, messages, tools, output parsers, and the `Runnable` protocol. It has minimal dependencies and is what partner packages build against. Everything else is meant to be swappable around this stable center.
- **`langchain`** — the main package: agents, high-level constructs, and re-exports. In the v1 line this is a comparatively thin layer.
- **Partner packages** (`langchain-openai`, `langchain-anthropic`, `langchain-google-genai`, …) — one per provider, versioned independently so a provider SDK bump doesn't force a core upgrade.
- **`langchain-community`** — the long tail of community-maintained integrations (hundreds of vector stores, loaders, tools). Broad but uneven in quality and maintenance.
- **`langgraph`** — a separate repository for stateful, graph-based agent orchestration. In v1, LangChain's agent runtime is implemented on top of it rather than duplicating control flow.

The historical composition primitive is **LCEL (LangChain Expression Language)** — the `Runnable` interface with a pipe operator (`prompt | model | parser`) that gives every composed unit uniform `invoke` / `batch` / `stream` / async methods. LCEL is elegant for linear pipelines but becomes awkward for branching, loops, and human-in-the-loop control, which is precisely the gap LangGraph was created to fill. Much of the older chain machinery that predates this split has, in the v1 reorganization, been moved out of the main package to a compatibility home rather than deleted, so migration is possible but not free.

The through-line: LangChain co-evolves aggressively with a fast-moving model landscape. Standardized tool-calling, structured output, and multimodal *content blocks* are added as providers ship them. That responsiveness is the reason the library stays useful — and the reason its surface keeps shifting.

## Production Notes

**Version churn is the headline operational cost.** The 0.x series shipped multiple reorganizations (the `langchain-core` / community / partner split around 0.1, the Pydantic 1→2 migration in 0.3[^3]), and v1 is another repositioning. Pin exact versions, read migration guides before upgrading, and budget for API drift. Blog tutorials go stale fast; trust the versioned docs.

**Dependency surface.** Installing `langchain-community` can pull a large transitive tree, and version conflicts *between* partner packages and `langchain-core` are a recurring source of dependency-resolution pain. Prefer installing only the specific partner packages you use rather than pulling in community wholesale.

**Leaky abstractions.** When a provider exposes a feature the abstraction doesn't model cleanly (a new parameter, a provider-specific response field), you end up reaching past the wrapper. Debugging often means tracing through several `Runnable` layers to find the real request. Enable verbose tracing early; this is where LangSmith is pitched, but it is a commercial product, not part of the OSS package.

**Streaming and async.** LCEL supports streaming and async, but behavior is not uniform across every integration — some community components silently fall back to sync or don't stream tokens. Verify streaming works end-to-end for *your* chosen model/vector store combination rather than assuming it.

**Prefer LangGraph for real control flow.** Multi-step agents, retries, branching, and human approval steps are better expressed in LangGraph than bolted onto chains. Teams that hand-roll complex control flow inside LCEL tend to migrate to LangGraph eventually; starting there for stateful agents avoids a rewrite.

## When to Use / When Not

**Use when:**
- You want to stay model-agnostic and swap providers without rewriting application code.
- You need breadth of integrations (many vector stores, loaders, tools) and value having them pre-wrapped.
- You're prototyping RAG or tool-using agents and want to move fast with batteries included.
- You'll also adopt LangGraph/LangSmith and want a coherent stack from one vendor.

**Avoid when:**
- Your app calls a single provider in simple ways — the provider's own SDK is thinner and more transparent.
- You need long-term API stability; LangChain's cadence of reorganizations is high.
- You want to see and control the exact request/response — the abstraction layers get in the way.
- Your orchestration is complex enough that you'll fight the chain model (use a graph/agent framework directly).

## Alternatives

- run-llama/llama_index — use instead when the app is retrieval/indexing-first; its data-framework abstractions fit RAG more naturally.
- langchain-ai/langgraph — same vendor; use directly when you need explicit stateful, graph-based agent control rather than chains.
- pydantic/pydantic-ai — use when you want a smaller, type-first agent framework with less abstraction and Pydantic-native structured output.
- crewAIInc/crewAI — use for role-based multi-agent collaboration with a higher-level, opinionated team metaphor.
- BerriAI/litellm — use when all you actually need is a unified API/proxy across providers, without the framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-10 | Harrison Chase publishes LangChain; "chains" + prompt utilities[^1]. |
| 0.1.0 | 2024-01 | First stable line; `langchain-core` + community/partner package split[^4]. |
| 0.2 | 2024-05 | Reduced `langchain` → `langchain-community` coupling; API cleanups. |
| 0.3 | 2024-09 | Internal migration to Pydantic 2[^3]. |
| 1.0 | 2025 | Repositioned as "agent engineering platform"; `create_agent`, standardized content blocks, thinner core over LangGraph[^2]. |

## References

[^1]: LangChain repository, created 2022-10-17. https://github.com/langchain-ai/langchain
[^2]: LangChain (Python) documentation overview. https://docs.langchain.com/oss/python/langchain/overview
[^3]: LangChain v0.3 migration notes (Pydantic 2). https://python.langchain.com/docs/versions/v0_3/
[^4]: "LangChain v0.1.0" — LangChain blog, 2024-01-08. https://blog.langchain.com/langchain-v0-1-0/

## Tags

python, llm, agents, ai-framework, rag, orchestration, generative-ai, langgraph, tool-calling, integrations
