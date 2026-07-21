# datawhalechina/hello-agents

A hands-on, from-scratch tutorial from the Datawhale community that teaches how AI agents work and how to build them, from single agents to multi-agent applications.

## What it is

Hello-Agents is a systematic learning course, not a runtime library. It walks readers from the theory of agents through practical construction, covering the core principles, classic paradigms, and system architecture behind LLM-driven agents. The material is written for people who already have basic Python skills and a conceptual grasp of large language models (for example, knowing how to call an LLM through an API), and it targets AI developers, software engineers, students, and self-learners. Its stated goal is to help readers move from being LLM "users" to becoming builders of agent systems, with a focus on AI-native agents rather than flow-driven, low-code software agents.

## Key features

- 16 chapters organized into five parts: agent and LLM fundamentals, building LLM agents, advanced topics, integrated case studies, and a capstone graduation project.
- Hands-on implementation of classic agent paradigms including ReAct, Plan-and-Solve, and Reflection.
- Coverage of both low-code platforms (Coze, Dify, n8n) and mainstream code frameworks (AutoGen, AgentScope, LangGraph), plus a walkthrough of building an agent framework from zero.
- A companion self-built framework, HelloAgents, constructed on top of OpenAI's native API (a separate repository, currently at V1.0.0).
- Advanced material on memory and retrieval (RAG), context engineering, inter-agent communication protocols (MCP, A2A, ANP), agent training with Agentic RL from SFT to GRPO, and performance evaluation.
- End-to-end case projects: an intelligent travel assistant, an automated deep-research agent, and a simulated "cyber town" of interacting agents.
- Full accompanying code is provided in the repository's `code` folder, and the book is available for free online reading and as a downloadable PDF.

## Tech stack

- Primary language: Python. The README expects readers to already have basic Python programming ability.
- No recognized root manifest (no root `requirements.txt`, `pyproject.toml`, or similar) is present; dependencies are scoped to individual example and community projects, several of which ship their own `requirements.txt` or `pyproject.toml`.
- The self-built HelloAgents framework is described as built on the OpenAI native API.
- Tooling discussed in the curriculum includes low-code platforms (Coze, Dify, n8n) and agent frameworks (AutoGen, AgentScope, LangGraph); these are taught as subjects rather than declared as project dependencies.
- Documentation is published as a static site (GitHub Pages plus a mainland-China mirror) and as PDF releases.

## When to reach for it

- You want a structured, sequential path to understanding how AI agents work rather than isolated code snippets.
- You have basic Python and want to implement paradigms like ReAct or Reflection yourself instead of only calling a high-level framework.
- You want to compare low-code platforms and code frameworks before committing to one, and to understand what building your own framework involves.
- You want coverage of advanced topics that many introductions skip, such as agent communication protocols, context engineering, Agentic RL training, and evaluation.
- You are learning in Chinese, or are comfortable with a Chinese-language primary text (an English README translation is linked).

## When *not* to reach for it

- You need a production-ready agent framework to drop into an application; this is educational material, and the actual framework (HelloAgents) lives in a separate repository.
- You want a language-agnostic or non-Python path; the examples and prerequisites assume Python.
- You need deep model-training or algorithm background as a starting point; the project deliberately centers on application and construction, so it is not a substitute for foundational ML theory.
- Your use case is flow-driven, low-code automation where the LLM is a backend step; the tutorial explicitly aims at AI-native agents instead of that style.
- Commercial reuse is a hard requirement; the content is licensed for non-commercial use (see Maturity signal).

## Maturity signal

Created in September 2025 and pushed as recently as mid-July 2026, the project has accumulated roughly 67,000 stars and 8,000 forks in well under a year, which signals rapid, sustained community adoption rather than a long-tail archive. It is not archived, all 16 chapters are marked complete, and the README describes an active roadmap (video courses, continued HelloAgents development, and a planned follow-up book), so it reads as actively maintained. The license is the notable caveat: the SPDX field resolves to NOASSERTION, and the README states Creative Commons BY-NC-SA 4.0, meaning attribution and share-alike are required and commercial use is not permitted.

## Alternatives

- Use the Hugging Face Agents Course (and smolagents) instead when you want an English-first, hands-on agents curriculum tied to the Hugging Face ecosystem.
- Use Microsoft's "AI Agents for Beginners" instead when you want a shorter, lesson-based English curriculum with runnable notebooks.
- Use LangGraph or AutoGen directly instead when you want to build on a maintained production framework rather than work through a from-scratch teaching repository.
- Use CrewAI instead when your goal is quickly assembling role-based multi-agent workflows rather than learning the underlying paradigms.

## Notes

- The maintainers deliberately watermark the free PDF with the Datawhale open-source logo to deter resellers from repackaging and selling it, while keeping the watermark unobtrusive for reading.
- The book draws an explicit distinction between "software-engineering" agents (Dify, Coze, n8n, where the LLM is a data-processing backend) and AI-native agents, and states that its focus is the latter.

## Tags

python, ai-agents, llm, rag, tutorial, multi-agent, agentic-rl, reinforcement-learning, mcp, machine-learning, educational
