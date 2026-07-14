# run-llama/llama_index

> Python framework for building LLM applications over private data — RAG-first, now repositioning around document agents and OCR.

[GitHub repo](https://github.com/run-llama/llama_index) ·
[Official website](https://developers.llamaindex.ai) ·
[License: MIT](https://github.com/run-llama/llama_index/blob/main/LICENSE)

## Overview

LlamaIndex began in late 2022 as **GPT Index**, Jerry Liu's project for indexing documents so an LLM could answer questions over them[^1]. It was renamed LlamaIndex in early 2023 and became one of the two dominant Python frameworks (alongside LangChain) for retrieval-augmented generation. Its original thesis was narrow and useful: connect data sources, build an index, query it with an LLM in a handful of lines.

Since 2024 the project has been pulled in two directions at once. The open-source framework has broadened from "RAG toolkit" to "agentic application framework" (its `Workflows` engine, multi-agent orchestration), while the company behind it — LlamaIndex Inc. — has pushed a commercial platform (LlamaParse / LlamaCloud) for agentic OCR and document extraction. The repository's own one-line description now reads "the leading document agent and OCR platform," language that describes the paid product more than the MIT-licensed library in this repo[^2]. Reading the wiki page for the framework, keep that framing gap in mind: the OSS package and the hosted platform are marketed together but licensed and billed apart.

The defining tension is **breadth versus stability**. LlamaIndex covers an enormous surface — 300+ integration packages for LLMs, embeddings, vector stores, and readers — and iterates fast enough that APIs deprecate within a year. The February 2024 v0.10 restructure (below) is the clearest example: a necessary cleanup that also broke nearly every existing install.

## Getting Started

```bash
# starter bundle: core + a curated set of integrations
pip install llama-index

# or install core and pick integrations explicitly
pip install llama-index-core llama-index-llms-openai llama-index-embeddings-openai
```

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# load ./data/*, chunk into Nodes, embed, and index (in-memory by default)
documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(documents)

query_engine = index.as_query_engine()
print(query_engine.query("What does the document say about pricing?"))
```

```python
# persist to disk and reload — avoids re-embedding on every run
index.storage_context.persist(persist_dir="./storage")

from llama_index.core import StorageContext, load_index_from_storage
index = load_index_from_storage(
    StorageContext.from_defaults(persist_dir="./storage")
)
```

## Architecture / How It Works

The data path is a pipeline of stable abstractions:

1. **Readers / connectors** (`SimpleDirectoryReader` and ~hundreds of `llama-index-readers-*` packages) load raw sources into `Document` objects.
2. **Node parsers** split Documents into `Node`s (chunks) with metadata and inter-node relationships.
3. **Indexes** organize Nodes. `VectorStoreIndex` is the common one; others include `SummaryIndex`, `DocumentSummaryIndex`, `KnowledgeGraphIndex`, and `PropertyGraphIndex`.
4. **Retrievers** fetch candidate Nodes for a query; **node postprocessors** rerank/filter them.
5. **Response synthesizers** stuff, refine, or tree-summarize retrieved Nodes into a final answer.
6. **Query engines / chat engines** wrap retrieval + synthesis into a callable interface.

Two cross-cutting pieces matter. **`Settings`** is a global object holding the default LLM, embedding model, tokenizer, and chunk size; it replaced the older `ServiceContext` in v0.10, and most tutorials predating that still show the dead API[^3]. **`Workflows`** is the newer event-driven orchestration layer: steps are async functions that emit and consume typed events, forming loops and branches — the intended replacement for the earlier `QueryPipeline` and ad-hoc agent runners for anything beyond a single query engine.

The **package topology** is the architectural fact with the largest blast radius. Since v0.10, `llama-index-core` contains the abstractions and every provider lives in its own PyPI package (`llama-index-llms-anthropic`, `llama-index-vector-stores-qdrant`, etc.) published from a separate monorepo and catalogued on LlamaHub. Imports encode this: `from llama_index.core.llms import LLM` is core; `from llama_index.llms.openai import OpenAI` is an integration package that must be installed separately. This keeps core lean but means a working environment is a specific set of independently-versioned packages that can drift out of sync with each other and with core.

## Production Notes

**The v0.10 migration is still the dominant upgrade scar.** Pre-0.10 code did `from llama_index import ...` and configured a `ServiceContext`; post-0.10 code imports from `llama_index.core`, installs integration packages, and uses `Settings`. Any tutorial, StackOverflow answer, or LLM-generated snippet from before February 2024 will fail on a current install. Verify import paths against current docs, not search results.

**Dependency resolution across integration packages is fragile.** Because each provider is versioned independently against a `llama-index-core` range, `pip` can assemble a set that installs but mismatches at runtime. Pin `llama-index-core` explicitly and add integrations against that pin; treat a fresh lockfile as a tested artifact, not an incidental one.

**Defaults are OpenAI, and they cost money silently.** `VectorStoreIndex.from_documents` embeds every chunk, and `as_query_engine()` calls an LLM — both default to OpenAI if a key is present. Large corpora can run up real embedding bills on first index. Set `Settings.llm` / `Settings.embed_model` before indexing, and persist the index so you embed once.

**In-memory by default does not scale.** The default vector store keeps embeddings in process memory and serializes to a local directory. For anything beyond a demo, back it with a real vector store integration (Qdrant, pgvector, Chroma, Weaviate, etc.); the in-memory store has no concurrency story and reloads the whole index into RAM.

**RAG quality is not free.** The five-line quickstart produces naive retrieval: fixed chunk size, top-k cosine, stuff-the-context synthesis. Production quality comes from the tuning surface — chunking strategy, hybrid/metadata retrieval, rerankers (node postprocessors), and query transformations. The framework exposes all of it, but the happy path does none of it.

**Framework overlap with the paid platform.** Document parsing in OSS (`SimpleDirectoryReader` + format-specific readers) is deliberately basic; high-fidelity parsing of complex PDFs is steered toward LlamaParse, a hosted paid API. That is a legitimate product boundary, but it means "just use LlamaIndex" for messy documents often implies a cloud dependency and per-page cost.

## When to Use / When Not

**Use when:**
- Your core problem is retrieval/QA over your own documents and you want indexes, retrievers, and synthesizers as first-class parts rather than plumbing you assemble.
- You need breadth of connectors and vector stores and value having a package for almost every provider.
- You're building document-centric agents and want `Workflows` plus tight RAG integration in one framework.

**Avoid when:**
- You want a small, stable dependency: the surface is large, releases are frequent, and major cleanups have broken imports within a single year.
- Your agents are tool/orchestration-heavy with little document retrieval — a leaner agent library fits better.
- You need fully offline, no-vendor-default behavior out of the box: the defaults assume OpenAI and, for hard documents, nudge toward the hosted parser.

## Alternatives

- langchain-ai/langchain — broader general-purpose LLM/agent framework; use it when orchestration and tool-use dominate and RAG is one part among many.
- deepset-ai/haystack — pipeline-oriented RAG/search framework with a more explicit, typed component graph; use it when you want production search pipelines over fast-moving abstractions.
- crewAIInc/crewAI — role-based multi-agent orchestration; use it when the problem is coordinating agents, not indexing documents.
- pydantic/pydantic-ai — typed, minimal agent framework; use it when you want structured outputs and type safety without a large RAG surface.
- microsoft/graphrag — graph-based RAG over a corpus; use it when relationship/entity-centric retrieval beats flat vector similarity.

## History

| Version | Date | Notes |
|---------|------|-------|
| GPT Index 0.x | 2022-11 | Initial public release; document indexing over LLMs[^1]. |
| Renamed LlamaIndex | 2023-early | Project and package rebranded from GPT Index. |
| 0.10.0 | 2024-02 | Major restructure: `llama-index-core` split from integration packages; `ServiceContext` deprecated in favor of `Settings`[^3]. |
| Workflows | 2024 | Event-driven agent orchestration introduced as successor to `QueryPipeline`/agent runners. |
| 0.12.x | 2024-late | Dropped older Python versions; continued Workflows/agent focus. |

## References

[^1]: Jerry Liu, LlamaIndex — original project (formerly GPT Index), repo created 2022-11-02. https://github.com/run-llama/llama_index
[^2]: LlamaIndex documentation portal (framework vs. LlamaCloud/LlamaParse platform). https://developers.llamaindex.ai
[^3]: LlamaIndex, "v0.10" migration and `Settings` (replacing `ServiceContext`). https://developers.llamaindex.ai/python/framework/

## Tags

python, rag, llm, agents, vector-database, retrieval, framework, document-processing, embeddings, ai
