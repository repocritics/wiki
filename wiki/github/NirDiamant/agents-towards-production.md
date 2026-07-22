# NirDiamant/agents-towards-production

A collection of hands-on, code-first tutorials for building and deploying production-grade GenAI agents.

## What it is

Agents Towards Production is an open-source set of runnable tutorials covering the building blocks of a production GenAI agent stack: orchestration, memory, retrieval, deployment, security, observability, evaluation, and UI. Each tutorial lives in its own folder with notebooks or code files and, in many cases, its own `requirements.txt`. It is aimed at developers who want to move agents from prototype to production using concrete, adaptable examples rather than prose documentation.

## Key features

- A stated 28+ production-grade tutorials, each self-contained and runnable.
- Tool integration examples including Model Context Protocol (MCP) and secure OAuth2 tool calling (Arcade).
- Memory tutorials using Redis (dual-memory and semantic search), Mem0 (hybrid vector and graph), and Cognee.
- Deployment paths covering Docker containerization, on-prem LLMs with Ollama, AWS Bedrock AgentCore, and GPU scaling on RunPod.
- Frameworks and serving via LangGraph, FastAPI endpoints, and a Kotlin agent with JetBrains Koog.
- Security and evaluation content, including LlamaFirewall guardrails and hands-on prompt-injection testing.
- Many tutorials are contributed by named sponsor companies, each demonstrating a specific tool.

## Tech stack

- Primarily Jupyter Notebooks, with Python as the main language.
- Dependencies are declared per tutorial via `requirements.txt`; there is no root manifest.
- Referenced tools and frameworks across tutorials: LangGraph/LangChain, FastAPI, Docker, Ollama, Redis, Mem0, Cognee, MCP, and the A2A protocol.
- At least one tutorial uses Kotlin (Koog) rather than Python.

## When to reach for it

- You want to add a specific production concern (memory, observability, security, deployment) to an existing agent.
- You prefer runnable, copy-and-adapt examples over conceptual documentation.
- You are evaluating a particular vendor tool in the context of an agent workflow.
- You are learning the end-to-end path from a prototype agent to a deployed one.

## When *not* to reach for it

- You want a library or framework to import — this is educational content, not an installable package.
- You want vendor-neutral coverage — many tutorials are sponsor-contributed and centered on a specific product.
- You need a maintained runtime dependency — the material is meant to be studied and adapted into your own code.

## Maturity signal

Created in mid-2025, the repository has around 21,000 stars, a last push in July 2026, and only 12 open issues, which fits an actively curated tutorial collection that receives steady additions rather than bug-driven churn. The license is reported as NOASSERTION, meaning GitHub did not detect a standard license, so the `LICENSE` file should be reviewed before any reuse. Overall this reads as an actively maintained, high-traffic educational resource.

## Alternatives

- The LangChain / LangGraph documentation and templates — use them when you want the framework's own first-party reference material.
- The OpenAI Cookbook or Anthropic cookbook — use them when you want provider-authored example code.
- An awesome-list style index of agent resources — use it when you want a link directory rather than runnable tutorials.

## Notes

- The license is NOASSERTION, unusual for a repository of this size, so the exact reuse terms are not machine-detected.
- The README is heavy on sponsor logos, and a large share of the tutorials are contributed by the sponsoring companies to showcase their own platforms.
- Although the repository is Python-notebook centric, it spans multiple runtimes, including a Kotlin (Koog) tutorial.

## Tags

python, jupyter-notebook, llm, agents, rag, tutorials, machine-learning, mcp, langgraph, observability, deployment
