# neovim/neovim

> A fork of Vim rebuilt around an async event loop and a scriptable RPC API — the editor most modern Vim-style tooling now targets.

[GitHub repo](https://github.com/neovim/neovim) ·
[Official website](https://neovim.io) ·
[License: Apache-2.0 (contributions since 2015) + Vim license (vim-patch code)](https://github.com/neovim/neovim/blob/master/LICENSE.txt)

## Overview

Neovim is a fork of Vim that began in 2014 as a refactoring effort, funded by a Bountysource campaign started by Thiago de Arruda, after a large architectural pull request was rejected upstream[^1]. The stated goals are narrow and unglamorous: simplify Vim's C codebase, split maintenance across multiple contributors, and expose the editor's internals so that UIs and plugins can be built without patching core. It is not a rewrite — Neovim still tracks and merges upstream Vim patches (marked with a `vim-patch` token), so it inherits Vim's modal editing model, Ex commands, and Vimscript wholesale.

The defining architectural bet is **decoupling**: the editor core runs an event loop and speaks a msgpack-RPC protocol, and everything else — the terminal UI, external GUIs, remote plugins, and integration clients in a dozen languages — is a client of that protocol. This is what let Neovim ship things Vim historically resisted: a first-class **Lua** runtime (config in `init.lua`, LuaJIT embedded), a built-in **LSP** client, and **Tree-sitter** parsing for syntax and structural editing. The cost is a fast-moving target — the plugin ecosystem, Lua API, and Tree-sitter parser ABI all change across minor versions, and Neovim's `0.x` versioning means "minor" releases routinely carry breaking changes.

Neovim is for developers who want a terminal, keyboard-driven, modal editor that is programmable in a real language, and who accept ongoing config maintenance as the price. Most users today reach it through a distribution (LazyVim, NvChad, AstroNvim, or the official `kickstart.nvim`) rather than configuring bare Neovim from scratch. GitHub's linguist labels the repo "Vim Script", but by lines of code the codebase is predominantly C and Lua.

## Getting Started

```bash
# macOS / Linux via package managers
brew install neovim          # Homebrew
sudo apt install neovim      # Debian/Ubuntu (often older)
# Or download a prebuilt binary from the Releases page (recommended for latest)
```

Neovim reads its config from `~/.config/nvim/init.lua` (XDG paths). A minimal Lua config wiring up a keymap and a built-in LSP server:

```lua
-- ~/.config/nvim/init.lua
vim.g.mapleader = " "
vim.opt.number = true
vim.opt.expandtab = true
vim.opt.shiftwidth = 2

-- a plain keymap: <leader>w to save
vim.keymap.set("n", "<leader>w", "<cmd>write<cr>", { desc = "Save file" })

-- attach the built-in LSP client to a language server (server must be installed)
vim.lsp.enable("lua_ls")   -- 0.11+ API; earlier versions use lspconfig setup

vim.api.nvim_create_autocmd("LspAttach", {
  callback = function(args)
    local buf = args.buf
    vim.keymap.set("n", "gd", vim.lsp.buf.definition, { buffer = buf })
    vim.keymap.set("n", "K",  vim.lsp.buf.hover,      { buffer = buf })
  end,
})
```

Existing Vim users can point Neovim at a `vimrc` (`:help nvim-from-vim`); Vimscript config is still interpreted natively.

## Architecture / How It Works

Neovim's core is C, built with CMake (a convenience Makefile wraps it). The pieces that distinguish it from Vim:

1. **libuv event loop.** The main loop was rewritten on top of libuv, giving Neovim non-blocking I/O for jobs, timers, and RPC channels. This is the change that made asynchronous job control and external processes tractable, and it underpins everything above it.
2. **msgpack-RPC API.** The editor exposes its state and operations over a msgpack-RPC protocol (`src/nvim/api`, `src/nvim/msgpack_rpc`). `nvim --embed` starts a headless instance that a parent process drives entirely over stdin/stdout. API functions are code-generated from C declarations, and the same API is what Lua, remote plugins, and GUIs call.
3. **Detached UI.** The built-in terminal UI (`src/nvim/tui`) is just one RPC client rendering `redraw` events. External GUIs — Neovide, VimR, and others — are separate processes attached to the same protocol. There is no in-process GUI coupling.
4. **Lua subsystem.** LuaJIT is embedded (`src/nvim/lua`); the `vim.*` module surface (`vim.api`, `vim.fn`, `vim.keymap`, `vim.loop`/`vim.uv`, `vim.treesitter`, `vim.lsp`) is the modern scripting interface. Because it is LuaJIT, the language is Lua 5.1 semantics — 5.3/5.4 features are unavailable.
5. **Vimscript subsystem.** Vim's scripting language is still implemented (`src/nvim/eval`) for compatibility, so the vast Vim plugin corpus keeps working.
6. **Providers.** Python, Ruby, and Node "remote plugins" run out-of-process and talk to core over RPC, rather than being linked into the editor as Vim does.

The **LSP client** (`vim.lsp`) and **Tree-sitter** integration are built into core rather than being plugins, but the useful surface around them (server configs, parser installation, completion UIs) still lives in the community plugin ecosystem — `nvim-lspconfig`, `nvim-treesitter`, `mason.nvim`, `nvim-cmp`/`blink.cmp`, and Telescope are near-universal in practice.

## Production Notes

**`0.x` versioning is not cosmetic.** Neovim has never shipped a 1.0. Each minor release (0.9 → 0.10 → 0.11) can and does deprecate or remove APIs. Configs and plugins pinned to one minor frequently break on the next. Teams standardizing on Neovim should pin the editor version, not just plugins.

**Tree-sitter parser ABI churn.** Tree-sitter parsers are compiled shared objects with an ABI that advances across Neovim versions. Upgrading Neovim can require rebuilding/reinstalling every parser; a mismatched parser fails at load. `nvim-treesitter` (and its rewritten `main` branch) manage this, but it is a recurring upgrade-day surprise.

**Plugin ecosystem velocity.** The churn that makes Neovim capable also makes it fragile. High-traffic plugins rewrite their APIs (e.g., completion and Tree-sitter tooling have had breaking rewrites), and a config that worked six months ago may need edits. This is the single biggest operational cost of running Neovim as a daily driver.

**Startup time.** Bare Neovim starts in tens of milliseconds; a plugin-heavy config without lazy-loading can reach hundreds. Lazy-loading (via `lazy.nvim`) is effectively mandatory for large setups. `nvim --startuptime` and profiling are the standard diagnostics.

**LuaJIT ceiling.** Being LuaJIT (Lua 5.1) is a performance win but a compatibility constraint: libraries written for later Lua versions do not run, and some numeric/string behavior differs from mainline Lua.

**`vim.loop` → `vim.uv` rename** and similar API migrations recur; deprecation warnings precede removal but the window is short by calendar standards. Watch `:help news` and `:checkhealth` on every upgrade.

**Distributions are the real UX.** Most production Neovim is a distribution config (LazyVim, NvChad, AstroNvim) plus local overrides. This is convenient but adds a layer whose upgrade cadence you must also track; debugging often means understanding both the distro's abstractions and bare Neovim underneath.

## When to Use / When Not

**Use when:**
- You want a terminal-native, modal, keyboard-driven editor programmable in Lua.
- You want built-in LSP and Tree-sitter as a foundation and are willing to assemble the ecosystem (or adopt a distribution).
- You are building an editor UI or tooling and want a scriptable, embeddable core (`nvim --embed`, msgpack-RPC).
- You already know Vim and want async, Lua, and a faster-moving core without leaving the modal model.

**Avoid when:**
- You want a stable, low-maintenance editor that rarely breaks on upgrade — mainline Vim or an IDE is calmer.
- You don't want to maintain config or debug plugin interactions; a batteries-included editor (VS Code, or Helix) fits better.
- You need a GUI-first experience with no terminal involvement.
- You require Lua 5.3/5.4 semantics in your editor scripting.

## Alternatives

- vim/vim — the upstream. More conservative, near-ubiquitous, minimal dependencies. Use when stability and availability everywhere matter more than async/Lua/LSP built in.
- helix-editor/helix — Rust, modal (selection-first), with LSP and Tree-sitter built in and TOML config. Use when you want batteries included without managing a Lua plugin stack.
- **kakoune** — modal editor with a multiple-selection interaction model and client/server architecture. Use when you prefer selection-then-action editing over Vim's verb-object grammar.
- emacs-mirror/emacs — extensible via Emacs Lisp with evil-mode for Vim keybindings. Use when you want a programmable environment beyond text editing.
- **VS Code** (with a Vim emulation extension) — GUI-first, large extension market. Use when you want IDE features and integration without terminal or config-as-code.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-02 | Project announced; Bountysource-funded fork of Vim[^1]. |
| 0.1.0 | 2015-11 | First tagged release. libuv event loop, msgpack-RPC API, embedded terminal[^2]. |
| 0.2.0 | 2017-06 | Stabilization; UI protocol and API improvements. |
| 0.3.0 | 2018-07 | Floating-window groundwork, API additions. |
| 0.4.0 | 2019-11 | Floating windows, extmarks, `wildoptions=pum`. |
| 0.5.0 | 2021-07 | Built-in LSP client, Tree-sitter integration, native Lua config[^3]. |
| 0.6.0 | 2021-11 | Lua as a first-class config language, `vim.keymap`, default colorscheme changes. |
| 0.7.0 | 2022-04 | Lua autocommands and user commands, `vim.filetype`. |
| 0.8.0 | 2022-09 | Extmark/decoration and UI refinements. |
| 0.9.0 | 2023-04 | Improved LSP and Tree-sitter, `:h` help enhancements, statuscolumn. |
| 0.10.0 | 2024-05 | Default Tree-sitter highlighting for select languages, OSC 52 clipboard, terminal and diagnostics improvements[^4]. |
| 0.11.0 | 2025-03 | `vim.lsp.enable` and streamlined LSP config, completion and UI updates. |

## References

[^1]: Neovim announcement and Bountysource campaign (Thiago de Arruda, 2014). https://neovim.io/news/2014/09/
[^2]: Neovim 0.1.0 release. https://github.com/neovim/neovim/releases/tag/v0.1.0
[^3]: "What's new in Neovim 0.5" — built-in LSP, Tree-sitter, Lua. https://neovim.io/news/2021/07/
[^4]: Neovim 0.10.0 release notes. https://github.com/neovim/neovim/releases/tag/v0.10.0
[^5]: Neovim documentation — `:help nvim-features`, API, and news. https://neovim.io/doc/

## Tags

text-editor, vim, modal-editor, c, lua, terminal, tui, developer-tools, extensibility, msgpack-rpc, lsp, tree-sitter
