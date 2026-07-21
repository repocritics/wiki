# ashishpatel26/500-AI-Agents-Projects

A curated catalog of 500+ AI agent projects and use cases, organized by framework and by industry, alongside a set of self-contained runnable agent examples.

## What it is

This repository collects and indexes AI agent implementations across the major frameworks (LangGraph, CrewAI, AutoGen, Agno, LlamaIndex) and across industries such as healthcare, finance, education, and cybersecurity. It combines a large curated table of external open-source projects with an `agents/` directory of roughly twenty self-contained runnable examples and a short CrewAI-plus-MCP course. The README addresses developers building their first or next agent, researchers surveying the landscape, teams evaluating frameworks, and students learning from real examples.

## Key features

- A curated index of 500+ agent use cases browsable by industry and by framework.
- An `agents/` directory of self-contained examples, each with its own `requirements.txt` and `.env.example`, described as needing no monorepo setup.
- A framework comparison table (LangGraph, CrewAI, AutoGen, Agno, LlamaIndex) with a quick decision guide.
- Industry-indexed tables linking each use case to an external GitHub implementation.
- A `crewai_mcp_course/` with lessons that build up to an MCP server example.
- A companion website published via GitHub Pages.

## Tech stack

- Primarily Python examples; each example bundles its own `requirements.txt` and `.env.example`, so dependencies are per-agent rather than repository-wide.
- A web front-end under `web/` built with Vite and React (`App.jsx`, `vite.config.js`, `package.json`), published through a Jekyll GitHub Pages workflow.
- No recognized root manifest, consistent with a curated collection rather than an installable package.
- Licensed MIT.

## When to reach for it

- Browsing agent implementations for a specific framework or industry.
- Running a minimal working agent quickly to learn a framework's shape.
- Comparing agent frameworks before committing to one for a project.
- Surveying the breadth of the agent ecosystem in one place.

## When *not* to reach for it

- When you need a maintained library or framework to build on; this is a catalog and set of examples, not a package with a root manifest.
- For production code, since the examples are demonstrations and many entries link to third-party repositories of varying maintenance.
- When you require guaranteed-working links, because external references can drift over time.

## Maturity signal

The project has over 34,000 stars and more than 6,000 forks, is MIT-licensed, was created in December 2024, and was last pushed in June 2026, which together point to an actively curated and widely referenced collection. Because it is primarily a link index and example set, "maintenance" here means ongoing curation and additions rather than a versioned, released codebase.

## Alternatives

- crewAIInc/crewAI-examples — use it when you only need CrewAI examples straight from the framework's own source.
- microsoft/autogen — use it when you want AutoGen's maintained notebook gallery rather than a cross-framework index.
- langchain-ai/langgraph — use it when you want LangGraph-first, canonical tutorials maintained by the framework authors.

## Notes

- Although GitHub reports Python as the primary language, the repository is largely a curated index; runnable code lives in per-example directories and in an external-link table.
- The examples in `agents/` are intentionally isolated, each carrying its own dependencies and environment template, so there is no shared install step.

## Tags

awesome-list, ai-agents, llm, python, multi-agent, agents, machine-learning, crewai, langgraph, autogen
