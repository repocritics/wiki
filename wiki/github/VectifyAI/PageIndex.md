# VectifyAI/PageIndex

A vectorless, reasoning-based RAG system that builds a hierarchical tree index from long documents and retrieves by having an LLM reason over that tree.

## What it is

PageIndex is a retrieval system for long professional documents that replaces vector similarity search with a table-of-contents-style tree index and LLM reasoning over it. Its premise is that similarity is not the same as relevance, and that reasoning over document structure yields better retrieval for documents that demand contextual understanding. It targets people working with financial reports, legal and regulatory filings, technical manuals, medical literature, and academic texts, where chunk-and-embed retrieval returns results that are similar but not relevant. Retrieval works in two steps: generate a tree-structure index, then perform reasoning-based retrieval through tree search.

## Key features

- Builds a hierarchical, table-of-contents-style tree index from long PDFs, with markdown input also supported via `--md_path`.
- Reasoning-based, agentic retrieval through tree search, with no vector database and no chunking.
- Traceable and explainable retrieval grounded in explicit page and section references.
- Context-aware retrieval that incorporates conversation history and domain knowledge.
- Command-line generation via `run_pageindex.py --pdf_path`, with options for model, table-of-contents page count, node size limits, and optional node IDs, summaries, and doc descriptions.
- Multi-LLM support through LiteLLM, plus an example agentic vectorless RAG demo built on the OpenAI Agents SDK.

## Tech stack

- Python.
- LiteLLM (`==1.84.0`) for multi-provider LLM access.
- PDF parsing via PyMuPDF (`==1.26.4`) and PyPDF2 (`==3.0.1`).
- python-dotenv (`==1.2.2`) and pyyaml (`==6.0.2`) for configuration.
- Optional `openai-agents` dependency for the agentic demo.
- Default model `gpt-4o-2024-11-20`.
- Licensed MIT.

## When to reach for it

- Retrieval over long professional documents such as financial reports, SEC filings, legal, regulatory, technical, medical, or academic material.
- Cases needing traceable, explainable retrieval tied to explicit sections and pages.
- Retrieval where relevance depends on reasoning and context rather than surface similarity.
- Building agentic RAG without standing up and maintaining a vector database.

## When *not* to reach for it

- Complex PDFs needing high-quality OCR, since the open-source repository uses standard PDF parsing and the README points to the cloud service for enhanced OCR.
- Short or simple documents where conventional vector RAG is cheaper and adequate.
- Markdown that was converted from PDF or HTML, which the README warns usually loses the original hierarchy; it recommends their OCR to convert first.
- Latency- or cost-sensitive workloads, since tree building and reasoning involve LLM calls per node.

## Maturity signal

PageIndex gathered over 34,000 stars in about a year, is MIT-licensed, was created in April 2025, and was last pushed in July 2026, indicating active development and rapid adoption. The open-source repository is positioned as the self-host tier of a larger commercial product that also offers a chat platform, cloud API, MCP integration, and enterprise deployment. The open-issue count (around 140) is worth noting relative to the project's age.

## Alternatives

- PageIndex cloud service (API and MCP) — use it when you need enhanced OCR and higher-quality tree building and retrieval than the standard PDF parsing in this repository, as the README notes.
- LlamaIndex — use it for conventional chunk-and-embed vector RAG when documents are short and similarity search is enough.
- LangChain — use it when you want a general-purpose RAG and agent framework rather than a focused vectorless tree index.

## Notes

- The README reports state-of-the-art 98.7% accuracy on the FinanceBench financial-QA benchmark via the Mafin 2.5 system built on PageIndex.
- The design is described as inspired by AlphaGo, applying tree search to document retrieval.
- PageIndex anchors a broader open-source ecosystem the README names: OpenKB, ChatIndex, ConDB, and a PageIndex MCP server.
- The open-source path uses standard PDF parsing, while the cloud service adds OCR and an enhanced pipeline.

## Tags

python, rag, retrieval-augmented-generation, information-retrieval, llm, pdf, document-processing, reasoning, agents
