# patchy631/ai-engineering-hub

A collection of end-to-end tutorial projects on LLMs, RAG, and AI agents, organized by difficulty and backed by runnable code.

## What it is

This repository is a learning hub of self-contained example projects covering large language models, retrieval-augmented generation, and real-world AI agent applications. The README groups the material into 93+ projects across three tiers — Beginner (22), Intermediate (48), and Advanced (23) — so that beginners, practitioners, and researchers can each find a starting point. It is a companion to the "Daily Dose of Data Science" newsletter, and its purpose is to teach AI engineering by giving readers working implementations to run, adapt, and extend rather than a single installable tool.

## Key features

- Projects sorted into Beginner, Intermediate, and Advanced tracks, each with concrete themes such as OCR/vision, chat interfaces, RAG, agents, voice/audio, MCP, and model evaluation.
- Retrieval examples spanning basic RAG (LlamaIndex + Ollama), multimodal RAG, agentic RAG with web fallback, and low-latency retrieval (for example, a Milvus + Groq project citing sub-15ms retrieval).
- Agent and workflow demos built on named frameworks including CrewAI, Microsoft AutoGen, and smolagents, plus Agent Communication Protocol (ACP) examples.
- A dedicated set of Model Context Protocol (MCP) projects, from custom MCP servers to MCP-powered agentic RAG and voice agents.
- Model-comparison and evaluation projects (for example, Llama 4 vs DeepSeek-R1, Qwen3 vs DeepSeek-R1) using observability tooling such as Opik.
- Advanced material on fine-tuning (DeepSeek with Unsloth), building reasoning models, and a from-scratch Transformer implementation, alongside production-oriented demos like a NotebookLM clone.
- An included AI Engineering Roadmap describing a learning path from Python to production AI.

## Tech stack

- Primary language reported as Jupyter Notebook; individual projects are predominantly Python.
- No recognized root manifest — the repository is a monorepo of independent project folders, each managing its own dependencies.
- Several projects use `uv` for dependency management, evidenced by per-project `pyproject.toml`, `uv.lock`, and `.python-version` files.
- User-facing app frontends built with Streamlit and Chainlit (per-project `app.py` files), with some configuration in YAML.
- A wide range of third-party services and libraries referenced across projects, including LlamaIndex, Ollama, Qdrant, Milvus, Groq, SambaNova, Cohere, AssemblyAI, Cartesia, Firecrawl, BrightData, GroundX, Zep/Graphiti, and Unsloth.
- Models exercised across projects include Llama 3.2/3.3/4, DeepSeek-R1 and Janus-Pro, Gemma 3, and Qwen 2.5/3, among others.
- License: MIT.

## When to reach for it

- You want runnable, self-contained reference implementations of RAG, agents, or MCP integrations to study or adapt.
- You are following a structured learning path and want projects graded by difficulty, from OCR apps up to fine-tuning and production systems.
- You want to compare how different models or frameworks behave on the same task using the included comparison projects.
- You are exploring a specific ecosystem (CrewAI, AutoGen, LlamaIndex, MCP) and want a concrete, working starting point rather than API docs alone.

## When *not* to reach for it

- You need a single installable library or framework — this is a tutorial collection, not a package with a stable public API.
- You want production-hardened, maintained software; the projects are teaching demos, and each folder pins its own dependencies without a shared build.
- You require consistency across projects; conventions, frameworks, and vendor services vary from folder to folder.
- You want minimal external dependencies — many projects rely on specific paid or hosted third-party services (for example AssemblyAI, BrightData, Firecrawl, GroundX, Cohere).

## Maturity signal

The repository is very widely followed (over 36,000 stars and 6,000 forks) and was last pushed in July 2026, less than two years after its October 2024 creation, which points to an actively maintained and rapidly growing resource rather than an abandoned or long-tail project. It is not archived, carries an MIT license, and has around 120 open issues — consistent with a busy, community-facing tutorial hub where new projects are added regularly. The high fork count and Trendshift badge suggest it is used heavily as a reference and starting point, but the star count reflects popularity as learning material, not production readiness of any individual project.

## Alternatives

- Use microsoft/generative-ai-for-beginners when you want a formal, lesson-structured course with exercises rather than a loosely organized project collection.
- Use openai/openai-cookbook when your work centers on the OpenAI API specifically and you want vendor-maintained recipes.
- Use mlabonne/llm-course when you want a curriculum focused on LLM fundamentals, fine-tuning, and theory rather than broad application demos.
- Use official framework example repositories (for example, LlamaIndex or CrewAI examples) when you want canonical, first-party samples for one framework kept in sync with its releases.

## Notes

- Despite listing Jupyter Notebook as the primary language, many of the higher-tier projects are Python applications with Streamlit or Chainlit interfaces rather than notebooks.
- The README advertises "93+ Production-Ready Projects," but the projects are structured as tutorials and demos; each is independently scoped and dependency-managed, so "production-ready" should be read as illustrative rather than a deployment guarantee.
- The repository doubles as newsletter marketing (a free eBook offer tied to the Daily Dose of Data Science mailing list), which is worth noting when evaluating it purely as a code resource.

## Tags

python, jupyter-notebook, machine-learning, llm, rag, ai-agents, mcp, fine-tuning, tutorial, examples
