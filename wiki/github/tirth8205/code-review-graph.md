# tirth8205/code-review-graph

A local-first code intelligence graph that gives MCP-compatible AI coding tools precise, blast-radius context so they read only the files that matter.

## What it is

code-review-graph parses a repository into an AST with Tree-sitter, stores it as a graph of nodes (functions, classes, imports) and edges (calls, inheritance, test coverage) in a local SQLite database, and queries it at review time to compute the minimal set of files an AI assistant needs to read. It exposes this to AI coding tools over the Model Context Protocol (MCP) and a CLI, tracking changes incrementally so the graph stays current. The problem it targets is AI tools re-reading large parts of a codebase for review tasks; the audience is developers using MCP-capable coding assistants on medium-to-large repositories and monorepos. Everything runs locally, with no source code sent to external services.

## Key features

- Blast-radius analysis: when a file changes, the graph traces callers, dependents, and tests that could be affected so the assistant reads only those.
- Incremental updates: diffs changed files via SHA-256 hashing and re-parses only what changed; the README cites a 2,900-file project re-indexing in under two seconds.
- Broad language coverage using Tree-sitter with targeted fallbacks — Python, JavaScript/TypeScript/TSX, Go, Rust, Java, C/C++, C#, Ruby, Kotlin, Swift, PHP, and many more, plus Jupyter/Databricks notebooks; framework-aware PHP/Laravel parsing is included.
- Custom languages without forking, via a `languages.toml` file mapping extensions to bundled grammars and node types.
- A composite GitHub Action that posts a sticky, risk-scored PR review comment with affected flows and test gaps, with an optional `fail-on-risk` merge gate, run entirely on the CI runner.
- A set of MCP tools plus CLI commands for building, updating, querying, traversing, semantic search, community detection (Leiden), execution-flow tracing, hub/bridge detection, exports (GraphML, Cypher, Obsidian, SVG), and a multi-repo watch daemon.
- Optional semantic search via vector embeddings (sentence-transformers, Google Gemini, or any OpenAI-compatible endpoint) and FTS5-based hybrid keyword search.

## Tech stack

- Primary language: Python, requiring `>=3.10`; package version 2.3.7, built with `hatchling`; classified as Development Status 4 - Beta.
- Core dependencies: `mcp>=1.0.0,<2`, `fastmcp>=3.2.4,<4`, `tree-sitter>=0.23.0,<1`, `tree-sitter-language-pack>=0.3.0,<1`, `pyyaml>=6.0,<7`, `networkx>=3.2,<4`, `watchdog>=4.0.0,<7`, and `tomli` on older Pythons.
- Local storage in SQLite under `.code-review-graph/`, with FTS5 full-text search; no external database or cloud service required for core graph storage.
- Optional extras: `sentence-transformers` and `numpy` (embeddings), `google-generativeai` (Google embeddings), `igraph` (communities), `jedi` (enrichment), `ollama` (wiki generation), matplotlib/pyyaml (eval).
- A separate VS Code extension written in TypeScript (esbuild), and console entry points `code-review-graph` and `crg-daemon`.

## When to reach for it

- Large repositories or monorepos where AI tools waste tokens re-reading unrelated files.
- Giving an MCP-compatible assistant (Claude Code, Codex, Cursor, Windsurf, and others the installer detects) targeted blast-radius context.
- Adding risk-scored PR reviews to CI without sending source code to any external service.
- Watching multiple repositories in the background via the daemon so graphs stay fresh without editor integration.

## When *not* to reach for it

- Small projects or trivial single-file changes, where the README notes graph context can exceed a naive file read because of the structural metadata overhead.
- Workflows depending on high-quality search ranking or flow detection, both flagged as current weaknesses (search MRR ~0.35; flow detection ~33% recall, weaker for JavaScript and Go).
- Tools without MCP support, unless you rely on the CLI, hooks, or daemon instead.
- Cases where you need the impact "recall 1.0" figure to mean true completeness — the maintainers state it is a graph-derived, circular upper bound.

## Maturity signal

The project grew fast, reaching over 24,000 stars within roughly five months of its creation, and it is under very active development with recent pushes and a steady release cadence (version 2.3.7). It is MIT-licensed and self-describes as Beta. Notably, the README is candid about limitations and about not overstating benchmark numbers. Read it as young, fast-moving, and actively maintained, with functionality that is real but still maturing in search ranking and flow detection.

## Alternatives

- Aider's repository map — use it when you want tree-sitter-based repo context built directly into a specific coding CLI rather than a standalone graph and MCP server.
- Sourcegraph — reach for it when you want hosted, large-scale code search and navigation across many repositories rather than a local per-repo graph.
- Serena — consider it when you want an LSP-driven coding-agent toolkit over MCP instead of a persistent structural graph (its tooling also appears in this repo).

## Notes

The README is unusually transparent about its own numbers: the headline token reduction is a median of roughly 82x across six repositories, with 528x called out explicitly as the best case (fastapi), not the typical result, and the impact-analysis recall of 1.0 described as a circular upper bound rather than genuine full recall. Benchmarks are made reproducible by pinning upstream SHAs, fixing the community-detector seed, and using deterministic CPU embeddings. The dependency pin `fastmcp>=3.2.4,<4` is justified in-manifest as needing Message-based prompts and specific CVE fixes.

## Tags

python, knowledge-graph, mcp, tree-sitter, static-analysis, code-review, ai-coding, cli, sqlite, graphrag
