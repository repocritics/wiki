# Graphify-Labs/graphify

A CLI and AI-assistant skill that maps a codebase and its docs into a queryable knowledge graph using local AST parsing, with no vector store.

## What it is

Graphify turns a folder of code, docs, SQL schemas, configs, PDFs, and other media into a knowledge graph you traverse instead of grepping through files. Code is parsed deterministically and locally with tree-sitter, so nothing leaves the machine for the code pass; only the optional semantic pass over docs and media calls a model backend. It installs as a `/graphify` skill for Claude Code, Cursor, Codex, Gemini CLI, and 20+ other assistants, and is aimed at developers who want their coding agent to answer architecture and cross-file questions from a real graph rather than a similarity search.

## Key features

- Local, deterministic code parsing via tree-sitter AST across roughly 40 languages, producing `calls` / `imports` / `inherits` / `mixes_in` edges resolved across files — with no LLM and no data leaving the machine.
- Every edge is tagged `EXTRACTED` (explicit in the source) or `INFERRED` (resolved by graphify), so you can tell what was read directly from what was derived.
- A real graph you traverse rather than an embedding index: `graphify query` returns a scoped subgraph for a plain-language question, `graphify path A B` traces how two things connect, and `graphify explain` describes one node.
- Three output artifacts per run: `graph.html` (interactive), `GRAPH_REPORT.md` (god nodes, surprising connections, suggested questions), and `graph.json` (the full graph).
- Leiden community detection with LLM-free labels; design rationale (`# NOTE:` / `# WHY:` comments, ADR/RFC citations) is extracted as first-class nodes linked to the code.
- Optional inputs beyond code — docs, PDFs, images, and video/audio — plus an MCP server (stdio or HTTP) exposing `query_graph`, `get_node`, `get_neighbors`, `shortest_path`, and PR tools.
- Git hook for auto-rebuild on commit, a graph merge driver so `graph.json` is never left with conflict markers, and a `.graphifyignore` that merges with `.gitignore`.

## Tech stack

- Python, requires 3.10+; distributed on PyPI as `graphifyy` (double-y) with the CLI command `graphify`; MIT licensed; version 0.9.22 in `pyproject.toml`.
- Core dependencies: networkx >=3.4, numpy >=1.21, rapidfuzz >=3.0, and a large set of pinned tree-sitter grammar packages (python, javascript, typescript, go, rust, java, c, cpp, ruby, c-sharp, kotlin, scala, php, swift, lua, zig, powershell, elixir, objc, julia, verilog, fortran, bash, json, and more).
- Optional extras for PDF, office (`.docx`/`.xlsx`), Google Workspace, video/audio (faster-whisper + yt-dlp), MCP (with starlette floored above CVE fixes), Neo4j/FalkorDB export, SQL/Postgres introspection, Leiden (graspologic, Python < 3.13), and several model backends (OpenAI, Gemini, Anthropic, Bedrock, Azure, Ollama).
- setuptools build backend; dev tooling includes pytest, ruff, pyright, bandit, and pip-audit.

## When to reach for it

- You want an AI assistant to answer codebase questions from a graph, preferring scoped queries over reading files or grepping.
- You need cross-file relationships (call/import/inherit) for a large, multi-language repo mapped deterministically.
- You want code analysis that stays fully local with zero LLM cost for the code graph itself.
- You want a graph you can commit to git so a whole team starts from the same map, refreshed automatically on commit.

## When *not* to reach for it

- You want semantic similarity retrieval over prose; graphify is explicitly not a vector index and traverses a graph instead.
- Your questions are answered mainly over docs, PDFs, or media rather than code — that path requires a configured model backend and is not the local, free part.
- You installed the package with plain `pip` on macOS/Windows, which the README warns can lead to `ModuleNotFoundError` because the skill resolves Python at runtime; `uv tool install` or `pipx install` are recommended to avoid it.
- You need a language not covered by the bundled tree-sitter grammars or the optional extractors.

## Maturity signal

The project is young but developing quickly: created in April 2026, pushed within days of this page's facts (July 2026), MIT licensed, with a high star count and a pre-1.0 version (0.9.22). The default branch is `v8`, the extractor and skill layout is broad, and the dependency set is carefully pinned with security-motivated floors, all of which read as active, fast-moving development rather than maintenance mode. A Y Combinator S26 badge and a hosted "always-on" product on the roadmap indicate the open-source CLI sits alongside a commercial effort.

## Alternatives

- **Microsoft GraphRAG** — use it when you want an LLM to build the graph and are comfortable with per-token extraction cost; graphify deliberately parses code without an LLM.
- **[mem0](https://github.com/mem0ai/mem0)** — use it when you want a general memory layer for an application rather than a code-and-docs knowledge graph; graphify benchmarks against it on LOCOMO recall.
- **supermemory** — another memory system graphify compares against; reach for it when your need is conversational/agent memory rather than repository structure.
- **A conventional embeddings/vector RAG store** — use it when semantic nearest-neighbor search fits better than deterministic graph traversal.

## Notes

- The official PyPI package is `graphifyy` with two y's while the command stays `graphify`; the README warns that other `graphify*` packages are unaffiliated and that `uvx graphify` fails because the package, not the command, must be named (`uvx --from graphifyy graphify install`).
- The graph build reportedly uses zero LLM credits, and benchmark scoring used a judge blind-validated against a second judge (reported 90.6% agreement, Cohen's kappa 0.81).
- A strict mode on Claude Code can block the first raw source read of a session and redirect it to the graph, then revert to a soft nudge for the rest of the session.

## Tags

python, knowledge-graph, graphrag, rag, tree-sitter, static-analysis, ast, cli, mcp, developer-tools, code-search, llm
