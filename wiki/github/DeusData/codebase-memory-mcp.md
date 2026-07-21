# DeusData/codebase-memory-mcp

A single-binary MCP server, written in C, that indexes codebases into a persistent knowledge graph for fast, token-efficient structural queries by AI coding agents.

## What it is

codebase-memory-mcp parses a codebase with tree-sitter and builds a persistent, SQLite-backed knowledge graph of functions, classes, call chains, HTTP routes, and cross-service links. It is a structural analysis backend, not an LLM: it answers structural queries and relies on the connected MCP client to be the intelligence layer. It targets developers using MCP-compatible coding agents who want graph queries in place of repeated file-by-file grep and read cycles. It ships as a single static binary for macOS, Linux, and Windows with no runtime dependencies.

## Key features

- Indexes 158 languages using vendored tree-sitter grammars compiled into the binary.
- Adds Hybrid LSP semantic type resolution for twelve languages: Python, TypeScript/JavaScript/JSX/TSX, PHP, C#, Go, C, C++, Java, Kotlin, Rust, and Perl.
- Exposes 15 MCP tools spanning search, call-path tracing, architecture overview, git-diff impact mapping, Cypher-like queries, dead-code detection, cross-service HTTP linking, and ADR management.
- Semantic search using bundled Nomic `nomic-embed-code` embeddings compiled into the binary, plus BM25 full-text search via SQLite FTS5 and structural regex search.
- Cross-service detection (HTTP, gRPC, GraphQL, tRPC, pub-sub channels), cross-repo linking, and infrastructure-as-code indexing for Dockerfiles, Kubernetes manifests, and Kustomize overlays.
- Optional 3D graph visualization UI at `localhost:9749`, and a team-shared graph artifact committed as a single compressed file so teammates can skip reindexing.

## Tech stack

- A pure C core with zero dependencies, statically linked, combining tree-sitter grammars, Aho-Corasick matching, LZ4 compression, in-memory SQLite, and FTS5.
- Bundled Nomic `nomic-embed-code` embeddings (768-dimension int8) compiled into the binary, so semantic search needs no API key, Ollama, or Docker.
- An optional graph UI built with TypeScript, React, and Vite under `graph-ui/`, shipped as a separate `ui` build variant.
- Persistence to `~/.cache/codebase-memory-mcp/` via SQLite; the team artifact is zstd-compressed.
- Distributed through npm, PyPI, Homebrew, Scoop, Winget, Chocolatey, AUR, and `go install`; building from source needs a C/C++ compiler and zlib.
- Licensed MIT.

## When to reach for it

- Giving an MCP coding agent fast structural understanding of a large codebase.
- Reducing token usage relative to file-by-file exploration; the README cites roughly 99% fewer tokens across five structural queries.
- Impact analysis on uncommitted changes, dead-code detection, call-path tracing, and one-call architecture overviews.
- Mapping cross-service and cross-repo architecture across an indexed fleet.
- Sharing a prebuilt graph artifact with teammates to avoid a full reindex.

## When *not* to reach for it

- When you expect a built-in LLM; this is a structural backend only and depends on your MCP client for natural-language reasoning.
- Workflows without an MCP-capable client, though a CLI mode exists for direct queries.
- Environments that cannot run an unsigned binary which also writes to agent configuration files; the README recommends auditing the signed, checksummed releases first.
- Questions about runtime behavior or intent that go beyond code structure.

## Maturity signal

The project reached over 33,000 stars since its creation in February 2026, was last pushed in July 2026, and is MIT-licensed, which points to very active and fast-moving development with rapid adoption. The high open-issue count (289) is consistent with a young project absorbing heavy usage. Releases are signed, checksummed, and scanned by multiple antivirus engines, and the design is described in an arXiv preprint (2603.27277).

## Alternatives

- graphify — use it if you prefer that workflow; the README explicitly compares its team artifact to graphify's `graphify-out/` directory.
- Single-language language servers (tsserver, pyright, gopls, rust-analyzer, and similar) — use them when you want full IDE-grade semantic analysis for one language rather than a cross-language graph built for agents; the README cites these as compatibility inspirations.
- Plain grep or ripgrep with file reads — use them for small codebases where the indexing overhead is not worth it, which is the baseline the README benchmarks against.

## Notes

- The entire engine is pure C with zero dependencies and ships as one static binary.
- Both the 158 tree-sitter grammars and the embedding model are compiled into the binary, so there is no external model download.
- There is deliberately no built-in LLM; the reasoning is that the MCP client already acts as the query translator.
- The team-shared artifact (`.codebase-memory/graph.db.zst`) auto-creates a `merge=ours` gitattributes line to avoid binary merge conflicts.
- Processing is entirely local with no telemetry, so performance and memory issues must be captured and shared manually through the documented diagnostics files.

## Tags

c, mcp, code-intelligence, knowledge-graph, tree-sitter, sqlite, static-analysis, developer-tools, cli, llm
