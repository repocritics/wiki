# headroomlabs-ai/headroom

A local-first context compression layer that shrinks everything an AI agent reads — tool outputs, logs, files, and RAG chunks — before it reaches the LLM.

## What it is

Headroom sits between your AI agent and the model provider and compresses the content flowing into the prompt so you pay for fewer input tokens without changing your answers. It targets developers running coding agents (Claude Code, Cursor, Codex, and similar) and applications that feed large tool outputs, logs, or JSON into an LLM. The reported savings range from roughly 15–20% on coding-agent workloads to 60–95% on structured JSON data, and compression is reversible so originals can be retrieved on demand. It ships in three shapes — an importable library (Python and TypeScript), a drop-in proxy, and an MCP server.

## Key features

- Content-aware compressors routed by type: SmartCrusher for JSON, an AST-aware CodeCompressor for several languages, and the Kompress-v2-base HuggingFace model for prose.
- Three integration modes — inline library (`compress(messages)`), an OpenAI-compatible proxy (`headroom proxy`), and an MCP server exposing `headroom_compress`, `headroom_retrieve`, and `headroom_stats`.
- `headroom wrap <tool>` configures and launches a supported coding agent through the proxy in one command, with `headroom unwrap` to undo durable wrapping.
- Reversible compression (CCR): originals are cached locally and the model can call a retrieval tool if it needs the full content within the configured TTL.
- CacheAligner stabilizes prompt prefixes and compresses only new bytes ("live zone") so provider KV caches keep hitting rather than being invalidated.
- Cross-agent shared memory with provenance and auto-dedup, plus `headroom learn`, which mines failed sessions and writes corrections to files like `CLAUDE.local.md`, `AGENTS.md`, or `GEMINI.md`.
- Optional output-token reduction from the proxy (verbosity steering and effort routing) for Anthropic and OpenAI-compatible endpoints, off by default.

## Tech stack

- Python 3.10+ package (`headroom-ai`), currently version 0.32.0, classified Development Status 4 - Beta.
- Rust workspace core built with maturin/pyo3 (0.29, abi3-py310); Rust edition 2021, `rust-version = 1.80`. Crates include `headroom-core` and `headroom-proxy`.
- Proxy stack: FastAPI (>=0.100.0), uvicorn (>=0.23.0,<1.0), httpx with HTTP/2, openai (>=2.14.0), and the Rust proxy using axum 0.7, tokio, and reqwest.
- Compression dependencies: tiktoken, pydantic (>=2.0), ast-grep-cli, tree-sitter / tree-sitter-language-pack for code, and ONNX Runtime plus transformers for the Kompress text model.
- MCP server via `mcp` (>=1.28.1); model registry and pricing via litellm (pinned for Python < 3.14).
- Memory and relevance: sqlite-vec by default, optional hnswlib (needs a C++ toolchain), fastembed for embedding relevance.
- Cloud provider signing in the Rust core: AWS SigV4 (`aws-sigv4`, `aws-config`) for Bedrock and `gcp_auth` for Vertex.
- Also published as a TypeScript SDK on npm (`headroom-ai`, library only — no CLI) and a Docker image on GHCR.

## When to reach for it

- You run AI coding agents daily and want token savings without editing your application code.
- You work across multiple agents (Claude, Codex, Gemini, Grok) and want a shared memory store with dedup.
- You feed large JSON, tool outputs, or logs into an LLM and want content-aware compression rather than naive truncation.
- You need compression that stays reversible, so the model can pull back the original bytes when required.
- You want a provider-agnostic proxy that any OpenAI-compatible client can route through with no code changes.

## When *not* to reach for it

- You use a single provider whose native prompt caching or compaction already meets your needs and you do not need cross-agent memory.
- You run in a sandboxed or restricted environment where a local proxy or helper process cannot start — Headroom is local-first and runs on your machine.
- You are on an unsupported platform combination: native wheels currently cover macOS Apple Silicon and Linux, and Intel macOS needs Docker or a manual ONNX Runtime setup.
- The heavier ML features expect specific toolchains (a C++ compiler for the HNSW backend, AVX2 on x86 for the ONNX paths), so minimal or older environments fall back to non-ONNX paths.

## Maturity signal

Headroom is very young — created in January 2026 and pushed on the same day this page was generated — yet it already carries an unusually large star count and hundreds of open issues, which points to rapid, high-volume adoption rather than a settled project. The pyproject still marks it Beta at version 0.32.0, so APIs and behavior are evolving (the README references retired components and internal "realignment" phases). Read the maturity as actively and heavily maintained but pre-1.0: expect ongoing change, and pin versions if you depend on specific compression behavior.

## Alternatives

- Use **LLMLingua** (Microsoft) when you want a focused prompt-compression research library and do not need a proxy, agent wrapping, or reversible retrieval.
- Use **LiteLLM** when your main need is a unified LLM gateway across providers with cost tracking and routing rather than content-aware compression — note Headroom itself uses LiteLLM for pricing.
- Use a provider's **native prompt caching / context compaction** (Anthropic, OpenAI) when you stay on one provider and its built-in mechanisms already reduce your token spend.

## Notes

- The `headroom` CLI ships only through the PyPI package; the npm `headroom-ai` is the TypeScript SDK and provides no CLI command.
- Output-token savings are reported as an honest estimate with a confidence interval because they are counterfactual; you can set a holdout percentage to get a measured number instead.
- The release build deliberately keeps stack unwinding (no `panic = "abort"`) so a single bad request cannot crash the long-lived proxy, and strips symbols to stay under PyPI's per-project storage ceiling.
- The dependency pins encode real supply-chain history: `ast-grep-cli` excludes 0.44.1 (a compromised release) and Python 3.14 support required marking litellm optional.

## Tags

python, rust, typescript, llm, token-optimization, compression, proxy, mcp, cli, library, context-engineering, rag
