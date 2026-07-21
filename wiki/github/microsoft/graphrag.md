# microsoft/graphrag

A modular, graph-based Retrieval-Augmented Generation pipeline that uses LLMs to extract knowledge graphs from unstructured text and reason over private data.

## What it is

GraphRAG is a data pipeline and transformation suite that extracts structured knowledge-graph data from unstructured text using LLMs, then uses that graph to improve retrieval-augmented answers. It is aimed at teams enhancing an LLM's ability to reason over private or narrative data where flat vector retrieval misses cross-document structure. The README explicitly frames the code as a demonstration of a methodology, not an officially supported Microsoft offering.

## Key features

- An indexing pipeline that turns unstructured text into a knowledge graph of entities, relationships, and community structures.
- Multiple query modes shown in the example notebooks, including global search, local search, DRIFT search, and dynamic community selection.
- A command-line workflow via `graphrag init`, `index`, `update`, `query`, and `prompt-tune`.
- Prompt tuning to adapt extraction prompts to your own data, which the README strongly recommends over out-of-the-box use.
- Migration notebooks to move indexes across major versions without full re-indexing.
- A modular workspace split into packages for chunking, storage, cache, vectors, input, and LLM access.

## Tech stack

- Python, constrained to `>=3.11,<3.14`, organized as a uv workspace monorepo under `packages/*`.
- Vector storage via LanceDB, visible in the example inputs, with Parquet files as pipeline artifacts.
- Task automation through poethepoet, versioning through semversioner, and documentation through mkdocs-material.
- Developer tooling includes ruff, pyright, and pytest (with asyncio and xdist).
- Distributed on PyPI as `graphrag`; licensed MIT.

## When to reach for it

- Question answering over a private corpus where global, cross-document questions matter more than single-passage lookups.
- Building an explainable knowledge-graph memory structure to augment LLM outputs.
- Narrative or complex datasets where similarity-only vector RAG misses thematic structure.

## When *not* to reach for it

- Cost-sensitive or small tasks, since the README warns that indexing can be an expensive operation and recommends starting small.
- Projects needing an officially supported, production-backed product, which the README explicitly says this is not.
- Simple similarity lookups where conventional vector RAG is sufficient and cheaper.
- Workflows needing stable configuration across upgrades, since minor version bumps may require re-running `init` and major bumps may require migration notebooks.

## Maturity signal

With over 34,000 stars, an MIT license, creation in March 2024, a last push in July 2026, and a long semversioner history running through the 3.x series, GraphRAG is very actively developed. That said, Microsoft frames it as a research demonstration rather than a supported offering, and warns about indexing cost and breaking changes between versions, so it is best treated as active but research-grade.

## Alternatives

- LlamaIndex — use it when you want general-purpose document indexing and RAG without committing to graph extraction.
- LangChain — use it when you need a broad orchestration framework rather than a focused graph-RAG pipeline.
- Neo4j with graph/vector retrieval — use it when you already run a graph database and want to manage the knowledge graph directly there.

## Notes

- The README carries an explicit warning that indexing can be expensive; reading the documentation and starting small is advised before running at scale.
- The project is presented as a methodology demonstration rather than a supported product, with a linked Microsoft Research blog post and an arXiv paper (2404.16130).
- Upgrading is not seamless: re-initialize configuration between minor versions and use migration notebooks between major versions.

## Tags

python, rag, knowledge-graph, llm, information-retrieval, machine-learning, graph, nlp
