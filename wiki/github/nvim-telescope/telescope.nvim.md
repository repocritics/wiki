# nvim-telescope/telescope.nvim

> A highly extendable fuzzy finder over lists for Neovim — pickers, sorters, and previewers, all in Lua.

[GitHub repo](https://github.com/nvim-telescope/telescope.nvim) ·
[License: MIT](https://github.com/nvim-telescope/telescope.nvim/blob/master/LICENSE)

## Overview

Telescope is the de facto fuzzy finder for Neovim. It presents any list — files, buffers, git commits, LSP symbols, help tags, quickfix entries — through a common three-window interface: a prompt where you type, a results list that filters live, and a preview pane. First released in 2020 after Neovim 0.5 shipped a real Lua runtime and built-in LSP, it became the reference example of what a "modern, Lua-native" Neovim plugin looks like[^1].

The design bet is modularity over speed. A picker is just a source of entries; a sorter scores those entries against the prompt; a previewer renders the highlighted one. All three are swappable, and the same machinery drives every builtin. This composability is why Telescope became a platform — there is a large ecosystem of extensions (`telescope-fzf-native`, `telescope-file-browser`, `telescope-ui-select`, project pickers, and hundreds more) that plug into the same picker API. It is the default finder shipped with most Neovim starter configs (LazyVim, kickstart.nvim, NvChad).

The defining tension is performance. Telescope is written entirely in Lua, and its default sorter is a Lua implementation of fuzzy matching. On large result sets (a monorepo's `find_files`, a big `live_grep`) the pure-Lua sort path is the bottleneck, which is why nearly every serious setup installs the compiled C sorter `telescope-fzf-native` as a near-mandatory dependency. Newer finders like fzf-lua exist specifically to trade Telescope's extensibility for raw throughput.

## Getting Started

Telescope requires Neovim built with LuaJIT and `plenary.nvim`[^2]. Install with a plugin manager (lazy.nvim shown):

```lua
{
  'nvim-telescope/telescope.nvim',
  version = '*',                          -- pin to the latest release tag
  dependencies = {
    'nvim-lua/plenary.nvim',
    { 'nvim-telescope/telescope-fzf-native.nvim', build = 'make' }, -- compiled sorter
  },
}
```

Bind the builtin pickers to keys:

```lua
local builtin = require('telescope.builtin')
vim.keymap.set('n', '<leader>ff', builtin.find_files, { desc = 'Find files' })
vim.keymap.set('n', '<leader>fg', builtin.live_grep,  { desc = 'Live grep' })
vim.keymap.set('n', '<leader>fb', builtin.buffers,    { desc = 'Buffers' })
vim.keymap.set('n', '<leader>fh', builtin.help_tags,  { desc = 'Help tags' })
```

Run `:checkhealth telescope` after install. `live_grep` and `grep_string` require [ripgrep](https://github.com/BurntSushi/ripgrep); `fd` and `nvim-web-devicons` are recommended.

## Architecture / How It Works

Every picker is an instance of the same pipeline, wired together in `pickers/init.lua`:

1. **Finder** — produces entries. A finder can be a static table (`buffers`), a oneshot job (`find_files` shelling out to `fd`), or a "dynamic" job that re-runs on every keystroke (`live_grep` re-invoking `ripgrep` with the new prompt).
2. **Sorter** — scores each entry against the current prompt and orders the results. The default `generic_sorter`/`fzy_sorter` is pure Lua. `telescope-fzf-native` replaces it with a compiled C matcher exposed through LuaJIT FFI.
3. **Previewer** — renders the highlighted entry. File previewers open a scratch buffer and attach Neovim's treesitter/syntax highlighting; term previewers stream a command's output (e.g. `git diff`).

The three panes are ordinary Neovim buffers and windows managed by a layout strategy (`horizontal`, `vertical`, `flex`, `dropdown`, `cursor`), so previews use the real editor's highlighting rather than a reimplementation.

**Entry manager and async.** As you type, the finder streams entries into an `EntryManager` that inserts them into the results buffer while the sorter assigns scores, all driven by plenary's coroutine-based async (`plenary.async`) on a libuv event loop. This is what keeps the UI responsive while a `ripgrep` job is still producing output — but it also means the scheduling and back-pressure logic lives in Lua, and heavy result streams contend with the editor's main loop.

**Configuration** is two-layered: a global `setup{ defaults = {...} }` block plus per-picker overrides passed as `opts` at call time (`builtin.find_files({ hidden = true })`). Extensions register through `telescope.register_extension` and become available under `require('telescope').extensions`.

## Production Notes

- **Install the native sorter.** `telescope-fzf-native` (compiled C, needs `make`/a compiler at build time) is treated as optional in the README but is effectively required for acceptable sorting latency on large lists. Without it, filtering thousands of entries is noticeably laggy. It is the single most impactful config change.
- **`live_grep` performance depends on ripgrep, not Telescope.** Each keystroke restarts an `rg` process; on huge trees, pass `--glob`/`additional_args` or use `grep_string`. There is no built-in debounce on some paths, so very fast typists can queue jobs.
- **Preview cost.** Treesitter highlighting in the preview pane is the usual culprit for slow scrolling through results. Large files can be excluded with a `preview.filesize_limit` / a custom `buffer_previewer_maker`, or previews disabled per picker (`previewer = false`).
- **Neovim version coupling is strict.** The maintainers test and support only the latest stable Neovim release and latest `HEAD` nightly[^2]. Telescope tracks core APIs closely; running an old Neovim against a new Telescope (or vice versa) surfaces breakage that will be closed as unsupported. Pin Telescope to a release tag and upgrade it alongside Neovim.
- **Breaking changes land on `master`.** There are release tags, but much of the ecosystem installs from `master`. Follow `doc/telescope_changelog.txt` before bumping; picker option names and action APIs have shifted over time.
- **Open issue count is high** (400+), characteristic of a large plugin sitting on a fast-moving editor core; many are environment-specific (ripgrep flags, terminal, Neovim version) rather than core defects. Triage is community-driven.
- **It is a UI toolkit, not just a finder.** Because pickers are generic, teams build custom pickers over arbitrary data (Jira tickets, k8s resources). That is a supported use, but you own the async and sorting characteristics of your source.

## When to Use / When Not

**Use when:**
- You want one consistent fuzzy-finding UI over files, grep, git, LSP, and custom lists.
- You value extensibility and a large ecosystem of ready-made pickers.
- You are on a current Neovim release and can add the native sorter.
- You want previews with the editor's real syntax/treesitter highlighting.

**Avoid when:**
- Raw throughput on very large repos is the priority — fzf-lua or a bare fzf integration will feel faster.
- You want a zero-dependency, minimal finder — Telescope pulls in plenary and benefits heavily from a compiled sorter and ripgrep.
- You are pinned to an old Neovim version you cannot upgrade.

## Alternatives

- ibhagwan/fzf-lua — wraps the compiled `fzf` binary; faster on huge lists, fewer moving parts. Use when performance beats extensibility.
- junegunn/fzf + fzf.vim — the original terminal fuzzy finder with a Vim/Neovim layer; use in Vim or when you already live in `fzf`.
- nvim-mini/mini.pick (part of mini.nvim) — minimal, dependency-light picker. Use when you want a small footprint over the extension ecosystem.
- folke/snacks.nvim (picker module) — modern all-in-one picker from the LazyVim author; use if you already adopt the snacks suite.
- Shougo/ddu.vim — highly configurable Vim/Neovim fuzzy UI; use when you want ddu's source/filter architecture.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-07 | Repository created; Lua-native finder for Neovim 0.5+[^1]. |
| 0.1.0 | 2022-02 | First tagged release; stabilized picker/action API. |
| 0.1.x | 2022–2025 | Ongoing tagged releases tracking Neovim core (LSP, treesitter, diagnostics). |
| master | 2026-06 | Actively maintained; latest push 2026-06-27, ~19.6k stars[^3]. |

## References

[^1]: telescope.nvim README, "What Is Telescope?" and Getting Started. https://github.com/nvim-telescope/telescope.nvim/blob/master/README.md
[^2]: telescope.nvim README, "Requirements" — supports only the latest stable and nightly Neovim, built with LuaJIT; depends on plenary.nvim. https://github.com/nvim-telescope/telescope.nvim#getting-started
[^3]: GitHub repository metadata for nvim-telescope/telescope.nvim (stars, forks, last push), fetched 2026-07-15. https://github.com/nvim-telescope/telescope.nvim

## Tags

lua, neovim, neovim-plugin, fuzzy-finder, picker, editor-tooling, developer-tools, cli-integration, ripgrep, extensible
