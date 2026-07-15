# neuml/txtai

> An embeddings-database framework that bundles vector search, an SQL/content store, graph, pipelines, workflows and agents into one Python library.

[GitHub repo](https://github.com/neuml/txtai) ·
[Official website](https://neuml.github.io/txtai) ·
[License: Apache-2.0](https://github.com/neuml/txtai/blob/master/LICENSE)

## Overview

txtai is an "all-in-one" AI framework built around an *embeddings database* — a
single object that unions a vector index (sparse and/or dense), a relational
content store, and a graph network[^1]. On top of that foundation it layers
pipelines (task-specific model wrappers), workflows (pipeline chains), agents,
and an HTTP/MCP API. It is developed by NeuML, a small company that also sells
consulting and a hosted offering around the same stack[^2].

The defining characteristic is scope. Where most of the ecosystem splits into
"vector database" (Qdrant, Weaviate, Chroma) versus "orchestration framework"
(LangChain, LlamaIndex), txtai deliberately packages both plus the model-serving
glue behind one Python API and one YAML config. This is the source of both its
appeal and its main tradeoff: a single `Embeddings` object with sensible
defaults gets you retrieval in a few lines, but the framework owns storage,
indexing, and model orchestration together, so you inherit its choices rather
than composing best-of-breed parts yourself.

It targets Python developers building semantic search and RAG who want to stay
local — models run in-process via Hugging Face Transformers and Sentence
Transformers, with no requirement to ship data to a remote API. It is Apache-2.0
licensed and built on Python 3.10+, Transformers, Sentence Transformers, and
FastAPI[^1].

## Getting Started

```bash
pip install txtai
```

The base install is intentionally light; pipelines, the API, graph, and some ANN
backends pull heavier optional dependencies (`pip install txtai[pipeline]`,
`txtai[api]`, `txtai[graph]`, and so on)[^3].

```python
import txtai

# An embeddings database; default model is a small sentence-transformer.
embeddings = txtai.Embeddings(content=True)
embeddings.index([
    "US tops 5 million confirmed virus cases",
    "Canada's last fully intact ice shelf has suddenly collapsed",
    "Beijing mobilises invasion craft along coast as tensions escalate",
])

# Natural-language query; content=True lets you also run SQL over metadata.
for uid, score in embeddings.search("climate change", 1):
    print(uid, score)
```

The same behavior can be exposed as a service with no code — a YAML config plus
`CONFIG=app.yml uvicorn "txtai.api:app"` starts a REST API[^1].

## Architecture / How It Works

The `Embeddings` class is the core. Internally it composes several swappable
subsystems:

- **Vector backend (ANN).** Dense vectors are stored in Faiss by default, with
  Hnswlib, Annoy, and a NumPy backend as alternatives selected by config[^4].
  The index is loaded into memory for search; persistence is explicit via
  `save()` / `load()` (or `.tar.gz` archives).
- **Content / relational store.** When `content=True`, document text and
  metadata are kept in an embedded database (SQLite by default, DuckDB
  optional), which is what enables hybrid `search("query", "SELECT ... WHERE
  ...")` queries combining vector similarity with SQL filtering[^1].
- **Sparse / hybrid.** A scoring module provides keyword/BM25 and sparse-vector
  indexes; hybrid search fuses sparse and dense scores. This is opt-in via
  config, not on by default.
- **Graph.** A semantic graph (NetworkX-backed) can be built over the indexed
  content for topic modeling, connectivity, and graph-path retrieval used by
  GraphRAG-style flows.

Above the database sit three orchestration layers. **Pipelines** wrap individual
model tasks (summarization, transcription, translation, labeling, LLM
generation) as callables. **Workflows** chain pipelines and arbitrary Python
into declarative, often YAML-defined, data flows. **Agents** are built on Hugging
Face's `smolagents` and can call embeddings, pipelines, workflows, and other
agents as tools; they work with any LLM txtai supports (local Transformers,
llama.cpp, or hosted OpenAI/Anthropic/Bedrock via LiteLLM)[^1].

The whole stack is config-driven: nearly every component can be declared in a
dictionary or YAML file, which is what lets the identical definition run as a
library import or as a served API (REST + MCP). The cost of that uniformity is
indirection — behavior is spread across config keys and default model choices
rather than explicit code.

## Production Notes

**Indexes are in-memory and single-node.** Faiss holds vectors in RAM; there is
no built-in sharding or clustering. Scaling past one machine's memory means
partitioning yourself or swapping in an external vector store. Budget RAM for
`num_vectors × dims × 4 bytes` plus overhead, and remember the content store
(SQLite) is a separate on-disk file from the ANN index.

**First run downloads models.** Default pipelines pull weights from the Hugging
Face Hub on first use — cold-start latency and disk usage can surprise you in
containers and CI. Pin model paths and pre-bake weights into images for
reproducible deploys. Set `HF_HOME` to control the cache location.

**Dependency weight is real.** `pip install txtai` is small, but the moment you
add LLM pipelines you pull `torch` + `transformers` (multi-GB), and different
extras (`ann`, `graph`, `pipeline-audio`, `api`) each add their own tree. Install
only the extras you use; audio/vision pipelines have native system prerequisites
(ffmpeg, libsndfile) that are easy to miss.

**GPU vs CPU is a manual choice.** Faiss ships as separate `faiss-cpu` /
`faiss-gpu` packages, and Torch device placement is yours to configure. There is
no automatic acceleration detection to rely on.

**Index format and version upgrades.** txtai has evolved its embeddings/index
format across major versions (notably the introduction of content storage and
sparse/graph features). Indexes written by an older major line are not always
forward-compatible; treat a major version bump as "re-index from source" unless
you have verified the specific migration path. Keep the raw documents so you can
rebuild.

**Defaults are demo-tuned.** The out-of-box embedding model is a small,
general-purpose sentence-transformer chosen for speed. It is fine for prototypes
but frequently not the right accuracy/latency point for production — pick a
domain-appropriate model rather than shipping the default.

## When to Use / When Not

**Use when:**
- You want local, private semantic search or RAG without standing up a separate
  vector database service.
- You value one Python API and one YAML config over assembling storage +
  orchestration + model-serving from separate projects.
- You need vector + SQL-metadata + graph retrieval in a single embedded object.
- You want to prototype an idea and later expose it as a REST/MCP service with no
  rewrite.

**Avoid when:**
- You need a horizontally scalable, multi-node vector store with replication and
  managed ops — use a dedicated database.
- Your team wants to compose best-of-breed components and treat vector storage,
  the LLM layer, and orchestration as independently swappable.
- Your corpus exceeds single-node memory and you don't want to build the
  partitioning yourself.
- You want the broadest third-party integration catalog for agents and tools.

## Alternatives

- langchain-ai/langchain — use when you need the largest integration/tooling
  surface for LLM apps and can absorb heavier, faster-moving abstractions.
- run-llama/llama_index — use when the focus is document ingestion and indexing
  for RAG with many data connectors.
- qdrant/qdrant — use when you need a standalone, horizontally scalable vector
  database with server-side filtering rather than an embedded library.
- chroma-core/chroma — use when you want a lightweight embedded vector store and
  will bring your own pipeline/LLM layer.
- deepset-ai/haystack — use when you want a comparable end-to-end NLP/LLM
  pipeline framework with a larger enterprise/production focus.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020-08 | Initial public release: similarity search over Transformer embeddings (Faiss)[^5]. |
| 4.0 | 2022 | Content storage — document text/metadata alongside vectors, enabling SQL filtering. |
| 5.0 | 2022 | Semantic graph; the "embeddings database" (vector + graph + relational) framing. |
| 6.0 | 2023 | Expanded LLM pipeline and RAG tooling. |
| 7.0 | 2024 | Sparse vectors and hybrid (sparse + dense) search. |
| 8.0 | 2024 | Agents built on smolagents; MCP API. |

(Version→date mapping above is compiled from general knowledge; confirm exact
release dates against the releases page[^5].)

## References

[^1]: txtai README and documentation, NeuML. https://github.com/neuml/txtai and https://neuml.github.io/txtai
[^2]: NeuML — company behind txtai. https://neuml.com
[^3]: txtai install guide (optional dependencies, minimal install, containers). https://neuml.github.io/txtai/install
[^4]: txtai ANN / vectors documentation. https://neuml.github.io/txtai/embeddings/configuration/ann
[^5]: txtai releases. https://github.com/neuml/txtai/releases

## Tags

python, semantic-search, vector-search, embeddings, rag, llm-orchestration, information-retrieval, nlp, vector-database, ai-agents, apache-2.0
