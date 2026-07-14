# deepset-ai/haystack

> Python framework for wiring LLM applications out of explicit, typed components — RAG, semantic search, and agents as inspectable pipelines rather than opaque chains.

[GitHub repo](https://github.com/deepset-ai/haystack) ·
[Official website](https://haystack.deepset.ai) ·
[License: Apache-2.0](https://github.com/deepset-ai/haystack/blob/main/LICENSE)

## Overview

Haystack is an open-source orchestration framework, maintained by the German company deepset, for building LLM applications in Python. It started life in 2019 as a neural search / question-answering toolkit built on top of Transformers and Elasticsearch — the extractive-QA era, before "RAG" was a common term[^1]. Its lineage traces to deepset's earlier FARM library, which is why the original 1.x PyPI package is named `farm-haystack`.

The current framework is a ground-up rewrite, Haystack 2.0, released in early 2024[^2]. The two lines are not source-compatible: 1.x (`farm-haystack`) and 2.x (`haystack-ai`) are effectively different frameworks that share a name and a philosophy. The defining idea of 2.x is that an application is a graph of **Components** — small objects with typed inputs and outputs — that you connect explicitly. Retrieval, prompt building, model calls, routing, and memory are all first-class nodes you can see, reorder, and swap, rather than steps hidden inside a monolithic chain.

The tension Haystack lives in: it trades the fast, magic, "one import and go" ergonomics of its competitors for explicitness. You write more wiring code, but the data flow is inspectable and serializable, which is the pitch for teams that have to run these systems in production and debug them later. deepset monetizes through Haystack Enterprise and a hosted deepset AI Platform, so the open-source project is the funnel for a commercial offering — worth knowing when reading roadmap priorities.

## Getting Started

```sh
pip install haystack-ai        # 2.x — the current framework
# NOT `pip install farm-haystack`, which is the legacy 1.x line
```

Integrations (model providers, vector stores) ship as separate packages:

```sh
pip install anthropic-haystack qdrant-haystack
```

A minimal RAG pipeline — components connected by named output→input edges:

```python
from haystack import Pipeline, Document
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

store = InMemoryDocumentStore()
store.write_documents([Document(content="Haystack is built by deepset.")])

template = "Given:\n{% for d in documents %}{{ d.content }}{% endfor %}\nQ: {{ query }}"

pipe = Pipeline()
pipe.add_component("retriever", InMemoryBM25Retriever(document_store=store))
pipe.add_component("prompt", PromptBuilder(template=template))
pipe.add_component("llm", OpenAIGenerator())

pipe.connect("retriever.documents", "prompt.documents")
pipe.connect("prompt.prompt", "llm.prompt")

result = pipe.run({"retriever": {"query": "Who builds Haystack?"},
                   "prompt": {"query": "Who builds Haystack?"}})
print(result["llm"]["replies"][0])
```

## Architecture / How It Works

The 2.x model has three core abstractions:

1. **Component** — a Python class decorated with `@component` that declares its output types with `@component.output_types(...)` and implements `run()`. Inputs are inferred from the `run()` signature. Components are the unit of reuse and the thing integrations ship.
2. **Pipeline** — a directed graph of components. Edges connect a named output socket to a named input socket, and the pipeline validates type compatibility at `connect()` time, not at runtime. Unlike a plain DAG, pipelines support branches, and loops (a component can feed back to an upstream one), which is what makes agent-style control flow and self-correction possible.
3. **Document Store** — the persistence interface behind retrievers. `InMemoryDocumentStore` ships in core; Elasticsearch, OpenSearch, Weaviate, Pinecone, Qdrant, pgvector, Chroma, Milvus and others live as separate integration packages.

Pipelines serialize to and from YAML, so a graph built in Python can be saved, version-controlled, and re-loaded — this is the basis for deploying pipelines as services and for deepset's tooling. Data moves between components as `Document` and `ChatMessage` dataclasses.

Agents in Haystack are built on the same substrate: an `Agent` / tool-calling loop is a component that repeatedly calls a chat generator, dispatches tool calls, and loops until a stop condition — the looping pipeline is the mechanism, not a separate engine. The `haystack-core-integrations` monorepo, versioned independently from core, is where the majority of provider and store connectors actually live[^3]. Serving pipelines over HTTP or MCP is done by a companion project, Hayhooks, rather than core itself.

## Production Notes

**The 1.x → 2.x migration is a rewrite, not an upgrade.** If you inherit a `farm-haystack` codebase, budget for reimplementation: node classes, the `Pipeline.add_node` API, and the old YAML schema have no drop-in equivalents. 1.x is in maintenance; new development targets 2.x only. Confirm every tutorial and Stack Overflow answer you find is 2.x — the ecosystem is full of 1.x examples that will not run.

**Version skew between core and integrations.** Because connectors live in separate packages on their own release cadence, it is possible for a core upgrade to outpace an integration, or vice versa. Pin both `haystack-ai` and each `*-haystack` integration explicitly in production.

**Telemetry is on by default.** Haystack sends anonymous component-usage events on initialization. Opt out with `HAYSTACK_TELEMETRY_ENABLED=False` (or the documented settings file) — relevant for air-gapped or compliance-sensitive deployments where outbound calls must be justified[^4].

**Type-checked connections are a real safety net, but move errors to wiring time.** Mismatched sockets fail at `pipeline.connect()`, which is good, but the messages are about socket types, not about your intent — expect to spend early debugging time reading what each component actually emits (`documents`, `replies`, `meta`) rather than what you assumed.

**Document store parity is uneven.** Feature support (metadata filtering syntax, hybrid search, async) varies by backend. Do not assume a filter that works against `InMemoryDocumentStore` behaves identically against Qdrant or OpenSearch; validate filters against the store you will actually deploy.

**`language: MDX` on GitHub is misleading** — it reflects the size of the docs website in the repo, not the codebase. Haystack is a Python framework (3.x).

## When to Use / When Not

**Use when:**
- You want RAG or agent pipelines whose data flow is explicit, inspectable, and serializable to YAML for deployment.
- You need to swap model providers or vector stores without rewriting application logic.
- A team has to operate and debug the system long-term and values structure over magic.
- You want branching / looping control flow (routing, self-correction, tool loops) as a first-class pipeline feature.

**Avoid when:**
- You want the largest possible integration catalog and community answer volume off the shelf — LangChain's ecosystem is bigger.
- You're doing a quick prototype and the explicit `connect()` wiring feels like ceremony.
- Your problem is index-centric data ingestion over many document formats — LlamaIndex is more specialized there.
- You need a .NET / TypeScript-first stack; Haystack is Python-only.

## Alternatives

- langchain-ai/langchain — use instead when you want the widest integration catalog and the most community examples, and can tolerate a looser, faster-moving abstraction.
- run-llama/llama_index — use instead when the hard part is data ingestion, indexing, and retrieval over heterogeneous sources rather than pipeline orchestration.
- microsoft/semantic-kernel — use instead when you're on .NET or need C#/Java-first agent orchestration.
- langchain-ai/langgraph — use instead when you want graph-based agent state machines but are already committed to the LangChain ecosystem.
- crewAIInc/crewAI — use instead when you specifically want a role-based multi-agent collaboration abstraction rather than a general pipeline framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2019-11 | Neural search / extractive-QA toolkit on Transformers + Elasticsearch[^1]. |
| 1.0 (`farm-haystack`) | 2022 | First stable line; node/pipeline API, reader/retriever QA focus. |
| 2.0 (`haystack-ai`) | 2024-03 | Ground-up rewrite: typed Components, explicit graph, LLM/RAG/agent focus[^2]. |
| 2.x ongoing | 2024–2026 | Agents, tool calling, multimodal, expanding integration monorepo[^3]. |

## References

[^1]: deepset / Haystack — project origins as a neural-search and question-answering framework. https://haystack.deepset.ai/overview/intro
[^2]: deepset, "Haystack 2.0" release announcement — the rewrite introducing the Component/Pipeline model. https://haystack.deepset.ai/blog/haystack-2-release
[^3]: deepset-ai/haystack-core-integrations — separately versioned monorepo of model-provider and document-store connectors. https://github.com/deepset-ai/haystack-core-integrations
[^4]: Haystack documentation, "Telemetry" — what is collected and how to opt out. https://docs.haystack.deepset.ai/docs/telemetry

## Tags

python, llm, rag, retrieval-augmented-generation, agents, orchestration, semantic-search, nlp, vector-search, framework, deepset
