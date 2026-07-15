# zed-industries/zed

> A GPU-rendered, Rust-native code editor built for input latency and real-time collaboration, from the people who made Atom and Tree-sitter.

[GitHub repo](https://github.com/zed-industries/zed) ·
[Official website](https://zed.dev) ·
[License: GPL-3.0-or-later (Apache-2.0 components)](https://github.com/zed-industries/zed/blob/main/LICENSE-GPL)

## Overview

Zed is a desktop code editor written in Rust by Zed Industries, a company founded in 2021 by Nathan Sobo and other members of the team that built GitHub's Atom editor and Tree-sitter[^1]. Its thesis is that the editor should feel as immediate as the developer's thinking: it renders its entire UI on the GPU through a custom framework (GPUI) rather than a browser engine, which is the direct reaction to Atom's Electron-induced sluggishness. The editor entered public preview on macOS in 2023, was open-sourced in January 2024, and added Linux support later that year[^2]. Windows builds exist but are, at the time of writing, community/preview-grade rather than an officially shipped download.

The two defining features are performance and multiplayer. Performance comes from the native GPU renderer, a rope/CRDT text buffer, and heavy use of Rust's threading — Zed is measurably fast at startup and keystroke-to-screen latency. Multiplayer means real-time collaborative editing (shared buffers, follow-cursor, channels, voice/screen share) with the same CRDT machinery that powers undo. This is the tension at the core of the project: the collaboration and AI features are genuinely differentiated, but they lean on Zed Industries' hosted backend, so an editor that is fully open source still routes some of its headline capabilities through a company-run cloud[^3].

Zed is a for-profit company's product that happens to be open source, not a foundation-governed community project. That shapes the roadmap (AI and collaboration get first-class investment) and the extension model (extensions are sandboxed WebAssembly, not first-class plugins with editor internals access).

## Getting Started

```bash
# macOS
brew install --cask zed
# Linux (official install script)
curl -f https://zed.dev/install.sh | sh
```

Zed is configured with JSON, not a scripting language. `~/.config/zed/settings.json`:

```jsonc
{
  "theme": "One Dark",
  "buffer_font_size": 15,
  "vim_mode": true,
  "format_on_save": "on",
  "languages": {
    "Rust": { "tab_size": 4 }
  },
  "lsp": {
    "rust-analyzer": {
      "initialization_options": { "check": { "command": "clippy" } }
    }
  }
}
```

Open a project with `zed .` from the shell (install the CLI via the command palette: `cli: install`).

## Architecture / How It Works

**GPUI** is the foundation: an in-house UI framework that lays out and paints every frame on the GPU. Rendering backends are platform-specific — Metal on macOS, and a GPU abstraction called Blade (Vulkan-backed) on Linux[^4]. Styling uses a Tailwind-like fluent API in Rust rather than CSS. GPUI is published under Apache-2.0 and is reusable outside Zed, though it is not a stable, documented public API.

**Text and collaboration.** Buffers are stored in a sum-tree (a self-balancing B-tree of text chunks) that makes large-file edits and range queries cheap. Edits are modeled as CRDT operations, which is what allows two people editing the same buffer to converge without a central lock — the same operation log also drives undo/redo. The collaboration server (`crates/collab`) is a separate Rust service, licensed AGPL-3.0, that relays these operations; the public download talks to Zed's hosted instance.

**Language intelligence** is split: Tree-sitter provides fast, incremental syntax trees for highlighting, structural selection, and outline; the Language Server Protocol provides diagnostics, completion, and rename. Zed ships Tree-sitter grammars and wires LSP servers per language, downloading them on demand.

**Extensions** are WebAssembly modules (typically Rust compiled to Wasm) loaded from the zed.dev registry. They can add languages (grammar + LSP glue), themes, slash commands, and context servers, but run in a sandbox — they cannot reach arbitrary editor internals the way a VS Code extension reaches the Node/DOM host. This is safer and more portable, and also strictly less capable.

**AI** is integrated as an assistant/agent panel and inline edit prediction. Prediction uses Zed's own open-weight model (Zeta); the assistant can use hosted Zed AI or your own provider API keys (Anthropic, OpenAI, Ollama, etc.). Agentic features and Model Context Protocol servers are wired through the same extension/context-server surface.

## Production Notes

- **The GPU renderer is a hard dependency.** Zed needs a working GPU driver and, on Linux, a Vulkan-capable stack. Headless servers, some VMs, remote X sessions, and older/exotic GPUs are common failure points; the editor may refuse to start or fall back poorly. This is the single most frequent Linux install issue.
- **Collaboration and hosted AI depend on Zed's servers.** The client is open source and the collab server source is available, but self-hosting `crates/collab` for your own team is not a documented, supported path. If Zed Industries' backend is unreachable, real-time collaboration and hosted-AI features are unavailable even though local editing is not.
- **Telemetry is on by default.** Zed collects usage and (opt-in-configured) crash/metric data. It is disableable in settings, but audit `telemetry` before deploying in privacy-sensitive orgs.
- **Extension ecosystem is small relative to VS Code.** Mainstream languages are covered, but long-tail tooling, debuggers, and niche integrations are frequently missing or thinner. The Wasm sandbox also means some VS Code-style extensions simply cannot be ported.
- **Windows is not first-class yet.** Treat Windows as preview: expect rougher edges than macOS/Linux and check the tracking issues before committing a team to it.
- **Config is JSON, not code.** No `.vimrc`-style programmability. Power users migrating from Neovim/Emacs will find the keymap and settings model comparatively closed, though Vim mode is capable and actively developed.
- **Debugger support arrived late.** DAP-based debugging is newer than the core editor; verify your language's debug adapter is supported rather than assuming parity with mature editors.

## When to Use / When Not

**Use when:**
- Input latency and startup speed are things you actually feel and care about.
- You want real-time pair/mob editing without a third-party plugin (Live Share equivalent, built in).
- You work primarily in mainstream languages (Rust, TypeScript, Python, Go, C/C++) on macOS or Linux.
- You want integrated AI (inline prediction + agent panel) with the option of bringing your own model keys.

**Avoid when:**
- You depend on a deep, niche VS Code extension or a specific debugger that Zed lacks.
- You run in headless/remote/VM environments where a GPU renderer is a liability.
- You need a fully self-hosted, vendor-independent collaboration stack with support guarantees.
- You want a scriptable, endlessly programmable editor (Emacs/Neovim) rather than JSON configuration.
- Windows is your primary platform and you need production stability today.

## Alternatives

- microsoft/vscode — the ecosystem default; use it when extension breadth, debugger coverage, or remote/container tooling matters more than native latency.
- neovim/neovim — use when you want a terminal-native, infinitely scriptable modal editor and are comfortable assembling your own config.
- helix-editor/helix — use when you want a batteries-included modal editor in Rust (Tree-sitter + LSP, no config) in the terminal.
- lapce/lapce — use when you specifically want a Rust GPU-accelerated editor with a native plugin system and don't need Zed's collaboration backend.
- zed-industries/zed's closest closed-source analog is Sublime Text — use that when you want a fast native editor without the AI/collaboration cloud dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021 | Zed Industries founded by ex-Atom / Tree-sitter team[^1]. |
| Preview | 2023 | Public preview on macOS. |
| Open source | 2024-01 | Editor open-sourced (GPL-3.0), GPUI under Apache-2.0, collab server AGPL-3.0[^2]. |
| Linux | 2024 | Official Linux builds released[^2]. |
| — | 2024–2025 | WebAssembly extension registry, AI assistant/agent panel, Zeta edit prediction, DAP debugging. |
| — | 2026 | Windows still preview; web version tracked but unshipped[^5]. |

## References

[^1]: Zed — "About / creators of Atom and Tree-sitter." https://zed.dev/about
[^2]: Zed Blog — "Zed is now open source" (2024-01-24). https://zed.dev/blog/zed-is-now-open-source
[^3]: Zed docs — Collaboration. https://zed.dev/docs/collaboration
[^4]: Blade GPU library used by GPUI on Linux. https://github.com/kvark/blade
[^5]: Web support tracking discussion. https://github.com/zed-industries/zed/discussions/26195

## Tags

rust, code-editor, ide, gpui, gpu-rendering, tree-sitter, lsp, real-time-collaboration, crdt, developer-tools, ai-assistant, desktop
