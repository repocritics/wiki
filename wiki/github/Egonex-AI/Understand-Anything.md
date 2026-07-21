# Egonex-AI/Understand-Anything

A tool that turns any codebase or knowledge base into an interactive knowledge graph you can explore, search, and question.

## What it is

Understand Anything analyzes a project with a multi-agent pipeline, builds a knowledge graph of every file, function, class, and dependency, and provides an interactive dashboard to explore it visually. It is aimed at developers facing large, unfamiliar codebases who need to see how the pieces fit together rather than read code blind. It installs as a Claude Code plugin and works across several other AI coding platforms.

## Key features

- A multi-agent pipeline (project scanner, file analyzer, architecture analyzer, tour builder, graph reviewer, plus domain and article analyzers) that extracts structure and produces graph nodes and edges.
- A Tree-sitter and LLM hybrid: Tree-sitter deterministically extracts imports, exports, definitions, and call sites, while the LLM adds plain-English summaries, layer assignments, and domain mapping.
- An interactive dashboard with structural and business-domain views, color-coded architectural layers, guided tours ordered by dependency, and fuzzy plus semantic search.
- Diff impact analysis, incremental updates that re-analyze only changed files, and an optional post-commit auto-update hook.
- The graph is plain JSON that can be committed and shared, and a standalone viewer (`npx`, Node >= 18) opens the dashboard read-only with no LLM calls or API key.
- Knowledge-base analysis for Karpathy-pattern LLM wikis and localized output in several languages (en, zh, zh-TW, ja, ko, ru).

## Tech stack

- TypeScript (`^5.7`), organized as a pnpm workspace monorepo (packageManager pnpm 10.6.2).
- Vitest (`^3.1`) for tests and ESLint (`^9`) with typescript-eslint for linting.
- Tree-sitter grammars for many languages (c, c#, c++, go, java, javascript, php, python, ruby, rust, scala, typescript) built via pnpm's `onlyBuiltDependencies`.
- An Astro-based homepage and a Vite dev server for the dashboard; sharp and ajv appear among build/runtime dependencies.
- The shareable viewer requires only Node.js (>= 18).

## When to reach for it

- Onboarding onto a large or unfamiliar codebase where you need a visual map of structure and dependencies.
- Sharing a committed architecture graph with a team so others skip the analysis pipeline.
- Reviewing the ripple effects of a change before committing via diff impact analysis.
- Working across multiple languages and wanting one navigable graph over all of them.

## When *not* to reach for it

- The codebase is small enough that reading it directly is faster than building a graph.
- The initial full analysis token cost is a concern on large repositories without a subscription or local model.
- You only need plain text or symbol search rather than an interactive graph.
- Your source language is not among the supported Tree-sitter grammars.

## Maturity signal

The project has gathered a very large star count within a few months of creation and was pushed on the snapshot date, signaling active, fast-moving development and broad adoption. It is MIT-licensed and not archived, with an open-issue count that is moderate for its size and reach. Continued multi-platform expansion and a monorepo with tests and benchmarks point to an actively maintained project still evolving quickly.

## Alternatives

- Sourcegraph — use it instead when you want large-scale code search and cross-repository navigation rather than an LLM-generated knowledge graph.
- Sourcetrail — use it instead when you want a purely deterministic interactive dependency explorer without LLM-generated summaries.
- DeepWiki — use it instead when you want an auto-generated browsable wiki for a repository rather than a graph dashboard you run and commit yourself.

## Notes

The design splits determinism from semantics on purpose: the structural side of the graph is reproducible because the same code always yields the same edges, while the LLM side captures intent that parsers cannot. Projects that already have a legacy `.understand-anything/` directory keep using it, and the newer default data directory is `.ua/`; large graphs are expected to be tracked with git-lfs.

## Tags

typescript, knowledge-graph, codebase-analysis, static-analysis, tree-sitter, llm, developer-tools, claude-code, code-visualization, multi-agent
