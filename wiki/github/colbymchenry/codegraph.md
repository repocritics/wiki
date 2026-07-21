# colbymchenry/codegraph

A local, pre-indexed code knowledge graph that gives AI coding agents surgical context in a single tool call instead of a grep-and-read crawl.

## What it is

CodeGraph builds a knowledge graph of every symbol, call edge, and dependency in a codebase and exposes it to AI coding agents through an MCP server. When an agent needs to understand or change code, it asks one question and gets back the relevant source, the call paths between symbols, and the impact radius of a change, rather than discovering structure file by file with grep, glob, and repeated reads. It targets developers using agents such as Claude Code, Cursor, Codex, opencode, Hermes Agent, Gemini CLI, Antigravity, and Kiro, and runs entirely on the local machine with a SQLite database and no external services.

## Key features

- One-call "surgical context": a single query returns entry points, related symbols, and code snippets, including dynamic-dispatch hops that grep cannot follow.
- Impact analysis that traces callers, callees, and the full blast radius of a symbol before a change.
- Full-text symbol search across the codebase, backed by SQLite FTS5.
- Auto-sync via a native file watcher (FSEvents, inotify, ReadDirectoryChangesW) with debounced re-indexing, plus a connect-time reconciliation so edits made while no server was running are absorbed on the next query.
- Structural extraction and cross-file resolution for 20+ languages into one graph with no per-language setup.
- Framework-aware route nodes across 17 web frameworks, and cross-language bridging for mixed iOS, React Native, and Expo codebases (Swift/ObjC, the RN bridge, TurboModules, Fabric, native-to-JS events, Expo Modules).

## Tech stack

- TypeScript, compiled with the TypeScript compiler (`typescript` ^5.0.0), published to npm as `@colbymchenry/codegraph` (version 1.4.1) with a `codegraph` CLI binary.
- Node.js runtime, `engines` pinned to `>=20.0.0 <25.0.0`; the standalone installer bundles its own runtime so no separate Node install is required.
- Parsing via tree-sitter WebAssembly grammars: `web-tree-sitter` ^0.25.3 and `tree-sitter-wasms` ^0.1.11.
- SQLite storage with FTS5 full-text search (`@types/better-sqlite3` in devDependencies).
- CLI and matching libraries: `commander` ^14.0.2, `@clack/prompts` ^1.3.0, `picomatch` ^4.0.3, `ignore` ^7.0.5, `jsonc-parser` ^3.3.1.
- Model Context Protocol (MCP) server as the integration surface for agents.
- Tests run with `vitest` ^2.1.9.
- GitHub reports C as the repository's primary language; the manifest also defines a `build:kernel` step (a bash script) and the test suite includes kernel-parity checks, indicating a native "kernel" component alongside the TypeScript codebase.

## When to reach for it

- You use an AI coding agent (Claude Code, Cursor, Codex, opencode, Gemini CLI, and others listed) and want it to answer architecture and navigation questions with fewer tool calls and near-zero file reads.
- You work in a large or tangled codebase where grep-and-read discovery burns time and tokens before real work starts.
- You need to see the impact radius of a change (callers, callees, affected flows) before editing.
- You have polyglot repositories, including mixed iOS/React Native/Expo projects, where call paths cross language boundaries that static parsing usually drops.
- You require a fully local setup with no data leaving the machine and no API keys.

## When *not* to reach for it

- You do not use an MCP-capable coding agent; CodeGraph's value comes through the agent integration, not standalone browsing.
- Your primary goal is token or dollar savings on a small codebase; the README states those savings are scale-dependent and small or noisy until repositories and teams get large, and speed is the reliable win instead.
- Your language or framework is outside the supported set; extraction quality depends on the tree-sitter grammars and framework detectors shipped.
- You need hosted, cross-organization code search over many repositories rather than a per-project local index.

## Maturity signal

CodeGraph is very actively developed. The repository was created in early 2026 and had a push the same day these facts were captured, the package is already at 1.x (README announces a 1.0 release and the manifest is at 1.4.1), and it carries a large test suite covering parsers, framework detectors, and the MCP daemon. Star and fork counts are unusually high for a project only months old, which signals rapid, broad interest more than a long track record; the 300-plus open issues are consistent with a fast-moving project under heavy real-world use rather than neglect. The MIT license and signed, attested npm releases lower adoption risk.

## Alternatives

- Sourcegraph: reach for it when you want hosted code search and navigation across an entire organization's repositories rather than a local per-project graph.
- Serena: reach for it when you prefer an MCP server that exposes language-server (LSP) semantics to agents instead of a precomputed graph.
- universal-ctags / GNU Global: reach for these when you just need a lightweight symbol index and can do without agent/MCP integration, cross-language bridging, or impact analysis.
- ast-grep: reach for it when you want structural code search and rewriting rules rather than an agent-facing knowledge graph.

## Notes

- Auto-sync is layered to avoid stale answers: during the debounce window, MCP responses prepend a warning banner naming a still-pending file and tell the agent to read it directly, so the agent is never silently given a wrong answer between an edit and the next sync.
- Cross-language bridge edges are tagged `provenance:'heuristic'` with a stable `metadata.synthesizedBy` channel name (for example `swift-objc-bridge`, `rn-event-channel`), so the agent can tell how a hop entered the graph.
- The published benchmarks report a consistent reduction in tool calls and roughly zero file reads, but the README is explicit that token and cost reductions are directional and move run to run, becoming a real line item only at large-codebase, high-volume scale.
- The installer only wires up agents; indexing a project is a separate `codegraph init`, and the graph lives in a local `.codegraph/` directory.

## Tags

typescript, nodejs, code-intelligence, knowledge-graph, static-analysis, mcp, developer-tools, cli, tree-sitter, sqlite, ai-agents, local-first
