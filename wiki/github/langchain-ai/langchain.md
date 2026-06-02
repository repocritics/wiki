# langchain-ai/langchain

The original "framework for LLM applications" — chains, agents, retrieval, memory, and tool integration primitives composed into one ecosystem. Now positioned as "the agent engineering platform" with LangGraph as the runtime layer.

## What it is

A Python (and JavaScript) library that defines abstractions for LLM-driven applications: model wrappers, prompt templates, output parsers, document loaders, vector stores, retrievers, memory buffers, agents, tools, and chains-of-everything. Authored by Harrison Chase, the project grew from a Jupyter notebook to a multi-language framework to a commercial-OSS company (LangChain Inc.). LangGraph is the newer runtime-graph layer that the modern positioning emphasizes over the legacy chains-and-agents abstractions.

## Key features

- Adapters for ~every major LLM provider (OpenAI, Anthropic, Google, Cohere, Hugging Face, local models).
- RAG primitives: document loaders for ~100 sources, text splitters, embedding stores, retrievers.
- Agent abstractions with tool calling, ReAct loops, and structured output.
- LangGraph — the graph-based runtime layer for multi-agent and multi-step workflows.
- LangSmith — proprietary observability / eval / tracing platform that integrates with the OSS framework.
- TypeScript companion (`langchainjs`) for Node/edge environments.
- MIT-licensed.

## Tech stack

- Python primary; TypeScript port is independently maintained as `langchainjs`.
- Pydantic for structured output + schema validation.
- Modular package structure (`langchain`, `langchain-core`, `langchain-community`, provider-specific packages) introduced in 2024 to address dependency-bloat criticism.
- Apache Arrow / Vector store integration for RAG.

## When to reach for it

- You want pre-built integrations for many LLM providers and data sources without re-implementing them.
- You're building agent workflows and want LangGraph's graph-execution primitives.
- You want the observability / eval surface (LangSmith) tightly coupled to your framework.

## When *not* to reach for it

- You want a minimal, code-first LLM integration — direct OpenAI/Anthropic SDK calls are lighter.
- You're allergic to framework churn — LangChain went through multiple major API rewrites between 2022-2024 (chains → LCEL → LangGraph).
- You want vendor-portable observability — LangSmith is the recommended path and is commercial.

## Maturity signal

138k stars, 23k forks, MIT, last push the day this page was generated. 3-year-old project that defined the modern LLM-framework category. The 607 open-issues count is moderate; the team has tightened triage as the project matured. The modular package split addressed historical complaints about dependency bloat. Commercial backing (LangChain Inc., LangSmith, LangGraph Platform) signals long-term viability.

## Alternatives

- DSPy — use when you want declarative-programming-style LLM pipelines with auto-optimization.
- LlamaIndex — use when RAG is the primary focus; better defaults for retrieval pipelines.
- `pydantic/pydantic-ai` — use when you want minimal, type-first LLM integration.
- Direct provider SDKs — use when you want zero abstraction overhead.

## Notes

The "framework churn" critique is real — most LangChain code written before mid-2024 needed rework to fit LCEL (LangChain Expression Language) and then again to fit LangGraph. The modern position is that LangGraph is the runtime and `langchain` is a collection of adapters; new projects should start with LangGraph rather than the legacy chain abstractions. License (MIT) is the safest among major LLM frameworks.

## Tags

artificial-intelligence, large-language-model, agent, python, typescript, framework, retrieval-augmented-generation, langchain, langgraph, openai, anthropic
