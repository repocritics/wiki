# microsoft/ai-agents-for-beginners

A Microsoft course of 18 lessons that teaches the fundamentals of building AI agents, with runnable Python code samples for each topic.

## What it is

This is an educational course, not a library or framework you install. It walks beginners through the concepts and practices of building AI agents across 18 self-contained lessons, each pairing a written README and short video with code samples. It is aimed at developers who are new to agentic AI and want a structured path from intro concepts to production, security, and deployment. The material is built around Microsoft's own agent tooling but assumes prior familiarity with generative AI models (the README points newcomers to the separate Generative AI for Beginners course first).

## Key features

- 18 lessons covering intro concepts, agentic frameworks, design patterns (tool use, planning, multi-agent, metacognition), Agentic RAG, trustworthy agents, context engineering, agent memory, production, deployment, local agents, and security.
- Each lesson bundles a written README, a short video, and code samples in a `code_samples` folder.
- Python code samples built on Microsoft Agent Framework (MAF) with Microsoft Foundry Agent Service V2; many lessons also include .NET/C# samples.
- Lessons are independent, so learners can start with any topic rather than following a fixed order.
- Automated multi-language support with 50+ translations generated and kept current via a GitHub Action (Co-op Translator).
- Lessons on agentic protocols (MCP, A2A, NLWeb), computer-use agents, and cryptographic receipts for securing agents.

## Tech stack

- Primary content format: Jupyter Notebook (course code samples), with additional C#/.NET samples.
- Python dependencies pinned in `requirements.txt`.
- Microsoft Agent Framework: `agent-framework~=1.10.0`, `agent-framework-foundry~=1.10.0`, `agent-framework-openai~=1.10.0` (pinned to the 1.10.x line; the manifest notes 1.11.0 introduced breaking API changes for the course notebooks).
- Azure services: `azure-ai-projects`, `azure-identity`, `azure-search-documents`; an Azure account is required for Microsoft Foundry.
- `openai>=1.108.1` (Responses API), plus `a2a-sdk` and `mcp[cli]` for agentic protocols.
- Lesson 17 local-agent stack: `foundry-local-sdk`, `chromadb`. Lesson 18 security stack: `jcs`, `pynacl`.
- Utilities: `httpx`, `ipykernel`, `nest-asyncio`, `numpy`, `pandas`, `pillow`, `python-dotenv`, `uvicorn`.
- Some samples support alternative OpenAI-compatible providers such as MiniMax; a devcontainer (`.devcontainer/`) is included.

## When to reach for it

- You are new to AI agents and want a structured, lesson-by-lesson curriculum rather than scattered tutorials.
- You want hands-on notebooks to learn agentic design patterns such as tool use, planning, multi-agent orchestration, and Agentic RAG.
- You are building on Microsoft's stack (Microsoft Agent Framework, Foundry Agent Service) and want vendor-aligned examples.
- You want reference code for specific topics such as agent memory, MCP/A2A protocols, computer-use agents, or securing agents.
- You need course material in a language other than English (50+ translations are maintained).

## When *not* to reach for it

- You need a production library or framework to depend on; this is a teaching repo, not a packaged product.
- You want a vendor-neutral path, since the samples center on Azure and Microsoft Foundry and generally require an Azure account.
- You are already experienced with agent development and want advanced or reference-grade architecture rather than beginner lessons.
- You need a fully local, no-cloud setup; only specific lessons (for example Lesson 17 with Foundry Local) cover local agents.
- You have not yet learned generative AI basics; the README recommends starting with Generative AI for Beginners first.

## Maturity signal

This is an actively maintained Microsoft educational repository. With roughly 70,000 stars and 23,000 forks, a recent push date, MIT license, and only a handful of open issues, it reflects a well-resourced, heavily adopted course rather than a stalled side project. The pinned framework versions and manifest comments about avoiding breaking changes in newer releases indicate the maintainers actively keep the notebooks working against a moving SDK. Because it is course content, "maturity" here means curriculum breadth and upkeep rather than API stability of a shipped library.

## Alternatives

- microsoft/generative-ai-for-beginners: use this instead when you need the prerequisite generative AI fundamentals before tackling agents (the README itself points beginners here first).
- microsoft/mcp-for-beginners: use this instead when your focus is specifically the Model Context Protocol rather than agents broadly.
- microsoft/langchain-for-beginners (and the LangChain.js / LangChain4j variants): use these instead when you want to learn agents through the LangChain ecosystem rather than Microsoft Agent Framework.

## Notes

- The repo carries 50+ language translations, which inflates its size; the README documents a sparse-checkout recipe to clone without the `translations` and `translated_images` directories for a faster download.
- `requirements.txt` deliberately pins Microsoft Agent Framework to 1.10.x and explains that 1.11.0 removed `ChatMessage` and `HostedWebSearchTool`, changed the `Message` constructor, and dropped the `model=` argument on `Agent.run()`, which would break the course notebooks.
- The repo ships an `.agents/skills` directory with reusable skills (for example Jupyter notebook authoring and an Azure OpenAI-to-Responses migration skill), suggesting the course is itself maintained with agent-assisted tooling.

## Tags

python, jupyter-notebook, ai-agents, agentic-ai, llm, rag, course, educational, tutorial, microsoft-foundry, semantic-kernel, mcp
