# microsoft/vscode

> Visual Studio Code — the open source core of the most widely used code editor.

[GitHub repo](https://github.com/microsoft/vscode) ·
[Official website](https://code.visualstudio.com) ·
[License: MIT](https://github.com/microsoft/vscode/blob/main/LICENSE.txt)

## Overview

`microsoft/vscode` is the open source core of Visual Studio Code, the cross-platform code editor Microsoft has shipped since 2015[^1]. The shipped product at `code.visualstudio.com` is a *superset*: it is this repository, plus closed-source Microsoft proprietary additions (telemetry, marketplace gallery client, Remote Development, Live Share, the C/C++ extension, etc.), packaged under a non-OSS license. Distributions like VSCodium[^2] strip out the Microsoft-proprietary additions and ship a pure-MIT build of the same source tree.

For wiki readers, the important distinction is that "VS Code", "Code - OSS", and "VSCodium" are not interchangeable. Extensions distributed via the Microsoft Marketplace are not licensed for use in non-Microsoft builds; this is a recurring source of confusion when Cursor, Windsurf, Trae, and other forks try to reuse them[^3].

## Getting Started

Most users install the proprietary binary; the OSS build requires compiling from source.

```bash
# Pure OSS build (Code - OSS) — large checkout, expects Node + Python
git clone https://github.com/microsoft/vscode
cd vscode
npm ci
npm run watch        # in one terminal
./scripts/code.sh    # in another (Linux/macOS)
```

```bash
# Pre-built OSS-equivalent
brew install --cask vscodium      # macOS
sudo apt install codium           # Debian/Ubuntu via paddatrapper PPA
```

For extension authors, the surface that matters is the `vscode` npm package types and the extension manifest:

```json
{
  "name": "my-extension",
  "engines": { "vscode": "^1.90.0" },
  "contributes": {
    "commands": [{ "command": "myExt.hello", "title": "Hello" }]
  },
  "activationEvents": ["onCommand:myExt.hello"]
}
```

## Architecture / How It Works

VS Code is an Electron application[^4] structured around three process boundaries:

1. **Main process** — Node.js, manages windows, IPC, and the lifecycle of renderer/extension processes.
2. **Renderer process** — Chromium, runs the UI (Monaco editor[^5], workbench, activity bar).
3. **Extension host** — separate Node.js process where third-party extensions run. This isolation is why a buggy extension can hang itself without crashing the editor; it's also why "main thread" UI APIs are not directly callable from extensions.

The editor surface itself is the Monaco editor — a separately published package that powers both VS Code and many in-browser editors (CodeSandbox, GitHub.dev, the OpenAI Playground, etc.).

Language intelligence is provided through the **Language Server Protocol (LSP)**[^6], which VS Code created with Eclipse and Sourcegraph in 2016 and which is now the de facto cross-editor standard. LSP servers are language-specific subprocesses; the editor itself knows nothing about TypeScript or Python. The **Debug Adapter Protocol (DAP)** is the analogous standard for debuggers[^7]. Both protocols are why Neovim, Helix, Zed, and most modern editors interoperate with the same tooling ecosystem.

The web variant (`github.dev`, `vscode.dev`) uses the same workbench code compiled for the browser, with the extension host running in a Web Worker. This restricts which extensions are usable — native binary extensions cannot run in the browser.

## Production Notes

**Memory profile.** A fresh window with one workspace folder is ~250–400 MB resident on macOS/Linux; large workspaces (>50k files) with TypeScript + ESLint + Prettier + GitLens routinely climb to 1.5–2 GB across all processes. The extension host is usually the dominant consumer.

**Startup latency.** Cold start is dominated by extension activation. Each extension declaring broad activation events (`onLanguage:javascript`, `*`) compounds startup time. The "Startup Performance" report (`Developer: Startup Performance`) is the standard diagnostic.

**Search & indexing.** VS Code uses `ripgrep`[^8] for in-workspace search. File watching on Linux uses inotify and can hit the system watch limit on large monorepos — the standard workaround is bumping `fs.inotify.max_user_watches`.

**Remote development.** The Remote-SSH, Remote-Containers, and WSL extensions are Microsoft-proprietary and not available in VSCodium builds. The OSS equivalents (`open-remote-ssh`, etc.) work but ship via OpenVSX[^9] rather than the Microsoft Marketplace.

**Extension marketplace lock-in.** Microsoft's Marketplace Terms restrict installation to Microsoft-branded products. Forks (Cursor, Windsurf, VSCodium) must use OpenVSX or sideload `.vsix` files. Several popular extensions (notably the C/C++ extension, Pylance, Remote Development, and Live Share) are not redistributable on OpenVSX. This is the most common production migration pain when leaving Microsoft's build[^3].

**Telemetry.** The OSS build does not include the telemetry endpoints by default; the shipped binary does, and the `telemetry.telemetryLevel` setting only suppresses certain categories. Air-gapped deployments typically standardize on VSCodium or a custom build to remove the telemetry binaries entirely.

## When to Use / When Not

**Use when:**
- You want a stable, well-supported editor with the largest extension ecosystem (~40k+ extensions on the Microsoft Marketplace).
- You need feature parity with the broader development tooling industry — LSP, DAP, devcontainers all originated here.
- You're building an editor or IDE and need a proven Electron + Monaco foundation.

**Avoid when:**
- You need a terminal-native, keyboard-driven editor for SSH sessions or low-resource environments: Neovim, Helix, Emacs, or micro are better suited.
- You can't ship Electron: the memory floor is high.
- You need true vendor neutrality: the proprietary additions and Marketplace restrictions create real lock-in even though the core is MIT.
- You want a Rust-native editor with built-in collaboration: Zed is a credible alternative.

## Alternatives

- Neovim (not yet in this wiki) — terminal-native, Lua-scripted, LSP/DAP-capable.
- Helix (not yet in this wiki) — terminal-native, Rust, modal, built-in LSP, no plugin system yet.
- Zed (not yet in this wiki) — Rust desktop editor with built-in collaboration, GPUI rendering.
- Emacs (not yet in this wiki) — older, more extensible, smaller default ecosystem for modern languages.
- Cursor / Windsurf / Trae — VS Code forks with embedded AI; share the same OSS base.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial preview | 2015-04-29 | Announced at Build 2015 as "Visual Studio Code"[^1]. |
| 1.0 | 2016-04-14 | First stable release. |
| LSP | 2016-06 | Language Server Protocol published[^6]. |
| Remote Development | 2019-05 | Remote-SSH / Containers / WSL (proprietary). |
| Live Share | 2017-11 | Real-time collaboration (proprietary). |
| `vscode.dev` | 2021-10 | In-browser build via Service Workers. |
| Profiles | 2022-12 | Settings + extensions grouped by profile (v1.75). |
| GitHub Copilot Chat OSS | 2025 | Microsoft moves Copilot Chat extension to MIT. |

## References

[^1]: Microsoft, "Visual Studio Code Announced at Build 2015" — 2015-04-29. https://devblogs.microsoft.com/visualstudio/announcing-visual-studio-code/
[^2]: VSCodium project. https://vscodium.com/
[^3]: VSCodium FAQ on Marketplace and proprietary extensions. https://github.com/VSCodium/vscodium/blob/master/docs/index.md
[^4]: Electron framework. https://www.electronjs.org/
[^5]: Monaco Editor. https://github.com/microsoft/monaco-editor
[^6]: Language Server Protocol specification. https://microsoft.github.io/language-server-protocol/
[^7]: Debug Adapter Protocol specification. https://microsoft.github.io/debug-adapter-protocol/
[^8]: BurntSushi/ripgrep. https://github.com/BurntSushi/ripgrep
[^9]: Open VSX Registry (Eclipse Foundation). https://open-vsx.org/
