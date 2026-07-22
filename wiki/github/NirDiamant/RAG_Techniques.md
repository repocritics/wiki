# NirDiamant/RAG_Techniques

A curated collection of runnable notebook tutorials covering retrieval-augmented generation techniques from foundational to advanced.

## What it is

RAG_Techniques is an educational repository, not a library. It gathers 42+ standalone Jupyter notebooks, each implementing and explaining a single RAG technique with its intuition, code, and references. It targets researchers and practitioners who want to learn and prototype specific retrieval and generation methods rather than adopt a single opinionated framework.

## Key features

- 42+ self-contained notebooks organized by category: foundational, query enhancement, context enrichment, advanced retrieval, iterative techniques, evaluation, explainability, and advanced architecture.
- Many techniques are provided in both LangChain and LlamaIndex variants, and several also ship as runnable Python scripts under `all_rag_techniques_runnable_scripts/`.
- Coverage of specific methods including HyDE, HyPE, semantic and proposition chunking, fusion retrieval, reranking, RAPTOR, Self-RAG, Corrective RAG (CRAG), and Graph RAG.
- A dedicated `evaluation/` directory with notebooks for DeepEval, GroUSE, End-to-End RAG Evaluation, and Open-RAG-Eval.
- Colab links for each notebook plus shared `helper_functions.py` and sample data under `data/`.

## Tech stack

- Primary language: Jupyter Notebook, with accompanying Python scripts.
- Uses LangChain and LlamaIndex as the two implementation ecosystems, and OpenAI for LLM and embedding calls.
- Individual notebooks pull in additional backends such as FAISS and Milvus for specific techniques.
- No root manifest is present, so there is no pinned dependency set; each notebook manages its own imports and requirements.

## When to reach for it

- Learning how a specific RAG technique works, end to end, from a documented reference implementation.
- Comparing LangChain and LlamaIndex approaches to the same retrieval problem side by side.
- Prototyping a retrieval method before committing it to a production codebase.
- Teaching or self-study on evaluation approaches for RAG systems.

## When *not* to reach for it

- Building a production system that needs an installable, versioned, and maintained package — this is a tutorial collection, not a dependency.
- Situations where a single supported framework with stable APIs is preferred over per-notebook code.
- Environments that cannot run notebooks or that require reproducible, pinned dependencies out of the box.

## Maturity signal

The repository is popular and actively curated, with a recent last push and a steady stream of newly added notebooks (recent additions include MemoRAG, End-to-End RAG Evaluation, Open-RAG-Eval, and JSON RAG). Its value is as a living catalog of techniques rather than a stable software artifact, so "maturity" here means breadth of up-to-date examples rather than API stability. Note that GitHub reports the license as NOASSERTION, meaning a standard SPDX license was not detected — check the repository's LICENSE file before reusing code in another project.

## Alternatives

- LangChain and LlamaIndex official documentation and templates — use these when you want maintained framework code with stable APIs rather than standalone technique notebooks.
- Microsoft GraphRAG — use it when you specifically need graph-based RAG packaged as a maintained library instead of a tutorial notebook.
- Haystack — use it when you want an end-to-end framework for building and deploying RAG pipelines rather than learning individual methods.

## Notes

The repository is a learning resource with no packaging layer: there is no root manifest, and each technique lives as an independent notebook (and sometimes a runnable script). Many techniques are duplicated across LangChain and LlamaIndex, so the raw notebook count overstates the number of distinct methods. The README also carries substantial course, book, and newsletter promotion around the technical content.

## Tags

python, jupyter-notebook, rag, retrieval-augmented-generation, llm, machine-learning, langchain, llama-index, tutorials, embeddings, semantic-search, nlp
