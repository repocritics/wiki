# folke/lazy.nvim

> A declarative, lazy-loading plugin manager for Neovim — the de facto standard since it displaced packer.nvim.

[GitHub repo](https://github.com/folke/lazy.nvim) ·
[Official website](https://lazy.folke.io/) ·
[License: Apache-2.0](https://github.com/folke/lazy.nvim/blob/main/LICENSE)

## Overview

lazy.nvim is a plugin manager for Neovim, first released in late 2022 by Folke Lemaitre, author of much of the modern Neovim plugin ecosystem (which-key, trouble, tokyonight, noice)[^1]. Within roughly a year it replaced packer.nvim — whose maintainer had stepped back — as the default choice for new Neovim configs, and it is the foundation the widely used LazyVim distribution is built on[^2]. At ~21k stars it is one of the most-depended-on pieces of the Neovim ecosystem despite being written entirely in Lua with no external dependencies beyond Git.

The defining idea is a **declarative plugin spec**: you describe plugins as Lua tables (source, dependencies, load triggers, config), and lazy.nvim resolves ordering, installs, and loads them. The second defining idea is aggressive **lazy-loading** — plugins can be deferred until an event, command, filetype, keymap, or `require()` of one of their modules actually fires. Startup only pays for what the current buffer needs.

The central tension is that lazy-loading is powerful but leaky. Deferring a plugin that expects to initialize at startup (colorschemes, statuslines, anything with global side effects) produces confusing "works after I open a file, not before" bugs. Much of the real-world lazy.nvim learning curve is understanding *when* a plugin can safely be deferred, not the spec syntax itself.

## Getting Started

Bootstrap lazy.nvim from your `init.lua` (it clones itself if missing):

```lua
-- init.lua
vim.g.mapleader = " "        -- MUST be set before require("lazy").setup()
vim.g.maplocalleader = "\\"

local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup({
  -- a plugin loaded on startup
  { "folke/tokyonight.nvim", priority = 1000, config = function()
      vim.cmd.colorscheme("tokyonight")
  end },

  -- lazy-loaded on a keymap; opts is passed to require("trouble").setup()
  { "folke/trouble.nvim", cmd = "Trouble", opts = {} },

  -- lazy-loaded on a filetype, with a dependency
  { "nvim-treesitter/nvim-treesitter", ft = { "lua", "go" }, build = ":TSUpdate" },
})
```

Requirements: Neovim >= 0.8.0 built with LuaJIT, and Git >= 2.19.0 (partial clone support)[^3].

## Architecture / How It Works

A plugin **spec** is a Lua table whose first element is the short GitHub `owner/name` (or a `dir`/`url` for local and non-GitHub sources). Common fields: `dependencies`, `event`, `cmd`, `ft`, `keys`, `opts`, `config`, `build`, `priority`, `version`/`branch`/`tag`, and `enabled`/`cond`. Specs can be split across many files under a `lua/plugins/` directory and merged via `import`, so a config is a tree of fragments rather than one list.

**Lazy-loading is handler-driven.** For each deferral trigger, lazy.nvim installs a lightweight stand-in that loads the plugin on first use and then replays the trigger:

- `event` — an autocmd. `VeryLazy` is a synthetic event fired after startup for "soon but not blocking" plugins.
- `cmd` — a stub user command that loads the plugin, then re-runs the real command.
- `ft` — an autocmd on `FileType`.
- `keys` — a placeholder keymap that loads the plugin then feeds the keystroke through.
- `module` — the most subtle one: lazy.nvim hooks Lua's `require` so that `require("some.module")` transparently loads the owning plugin on first access. This is why plugins used only as libraries need no explicit trigger.

**Startup performance** comes from two things. Partial clones (`--filter=blob:none`) fetch history lazily instead of a shallow snapshot, so `git log`/version resolution still works. And Lua modules are byte-compiled and cached; on Neovim 0.9+ this integrates with the built-in `vim.loader`. A `lazy-lock.json` lockfile records the exact commit of every plugin so installs are reproducible across machines.

Internally the codebase is organized into `core` (spec parsing, the plugin loader, the module cache, fragment merging), `manage` (async Git tasks and process orchestration for install/update/clean), and the floating-window UI. Operations are async — updates run concurrently rather than blocking the editor.

## Production Notes

**Set `mapleader` before `setup()`.** `keys` specs capture the leader at definition time; setting `vim.g.mapleader` after `require("lazy").setup()` silently maps the wrong prefix. This is the single most common misconfiguration.

**Colorschemes and other startup-critical plugins.** A colorscheme has nothing to lazy-load *on*, so give it `lazy = false` and a high `priority` (e.g. 1000) so it loads before other start plugins. Deferring statuslines, dashboards, or anything that must paint the first screen produces flicker or "no theme until I open a file" reports.

**`opts` vs `config`.** `opts = {...}` is merged and passed to the plugin's `setup()` automatically; `config = function() ... end` runs your own code and, if you also set `config = true`, calls `setup()` with no args. Mixing them up — e.g. providing `config` that never calls `setup()` — leaves a plugin installed but never initialized. Prefer `opts` unless you need imperative setup.

**`build` steps run on install/update, not startup.** Plugins needing compilation (nvim-treesitter's `:TSUpdate`, telescope-fzf-native's `make`) declare `build`. Forgetting it yields a plugin that loads but errors at runtime.

**Lockfile discipline.** `lazy-lock.json` is meant to be committed. `:Lazy update` rewrites it; `:Lazy restore` pins back to it. In team/dotfiles setups the lockfile is a frequent merge-conflict source, and updating it is a deliberate act, not automatic.

**Migration from packer.** The spec is similar but not identical: packer's `use`/`requires`/`run` map to lazy's table/`dependencies`/`build`, and packer's compiled `packer_compiled.lua` has no analogue (lazy caches transparently). Configs ported mechanically often over-lazy-load and hit the startup-side-effect bugs above.

**Version floor drift.** The minimum is Neovim 0.8, but the plugin ecosystem lazy.nvim manages increasingly assumes 0.10/0.11 APIs; in practice you want a recent Neovim, and LazyVim's own floor is higher than lazy.nvim's.

## When to Use / When Not

**Use when:**
- You maintain a Neovim config in Lua and want reproducible installs plus fast startup.
- You want plugins organized across multiple files with automatic merging.
- You're adopting LazyVim or another lazy.nvim-based distribution.
- You care about measured startup time (`:Lazy profile` gives a per-plugin breakdown).

**Avoid when:**
- You're on an old Neovim (< 0.8) or a build without LuaJIT.
- You want zero abstraction — `packadd` with Neovim's native package path and no manager is viable for tiny configs.
- Your editor is Vim, not Neovim (lazy.nvim is Neovim-only; use vim-plug or packer-for-vim there).

## Alternatives

- wbthomason/packer.nvim — the previous default; effectively unmaintained now, migrate off it.
- savq/paq-nvim — minimal, no lazy-loading, few features; use when you want the smallest possible manager.
- echasnovski/mini.nvim (mini.deps) — plugin management as one module of a larger suite; use if you already live in mini.nvim.
- junegunn/vim-plug — the classic; use for Vim or cross-Vim/Neovim configs where lazy.nvim's Neovim-only stance is a blocker.
- Neovim native `packadd` + `vim.pack` — Neovim 0.12 ships a built-in package manager; use when you want no third-party dependency and don't need lazy.nvim's loading handlers.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial release | 2022-11 | First public version by Folke Lemaitre[^1]. |
| Ecosystem shift | 2023 | Becomes the default choice for new configs as packer.nvim winds down. |
| LazyVim | 2023 | Folke's LazyVim distribution ships on top of lazy.nvim, driving adoption[^2]. |
| `stable` branch + luarocks | 2023–2024 | `--branch=stable` bootstrap and optional rockspec/luarocks package source. |
| Ongoing | 2026-06 | Actively maintained; last push mid-2026, ~21k stars, 66 open issues[^3]. |

## References

[^1]: folke/lazy.nvim repository and release history. https://github.com/folke/lazy.nvim/releases
[^2]: LazyVim — Neovim distribution built on lazy.nvim. https://www.lazyvim.org/
[^3]: lazy.nvim documentation (requirements, installation, spec reference). https://lazy.folke.io/

## Tags

lua, neovim, neovim-plugin, plugin-manager, package-manager, lazy-loading, editor-tooling, dotfiles, developer-tools
