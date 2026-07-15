# nvim-treesitter/nvim-treesitter

> Parser installation and query files that wire Tree-sitter into Neovim — highlighting, folds, indentation, and injections for hundreds of languages.

[GitHub repo](https://github.com/nvim-treesitter/nvim-treesitter) ·
[License: Apache-2.0](https://github.com/nvim-treesitter/nvim-treesitter/blob/main/LICENSE)

## Overview

nvim-treesitter is the plugin that made Tree-sitter usable in Neovim for ordinary
users. Tree-sitter itself is an incremental parser-generator that produces a
concrete syntax tree for source code[^1]; on its own it ships neither compiled
grammars nor the query files that map tree nodes to editor behavior. This plugin
fills both gaps: it installs and pins per-language parsers (compiled C extensions
generated from Tree-sitter grammars) and ships the `.scm` query files that drive
highlighting, folding, indentation, and language injection[^2].

Its history is a slow handover to Neovim core. When the project started (2020), the
Tree-sitter runtime in Neovim was experimental and the plugin carried its own module
system — `highlight`, `incremental_selection`, `indent`, and more — implemented in
Lua on top of that runtime. Over successive Neovim releases the runtime stabilized and
absorbed most of that surface (`vim.treesitter.start()`, `foldexpr`, `:InspectTree`),
which left the plugin's original job — bundling parsers and queries — as the part that
still had to live outside core. The current `main` branch is a full, deliberately
incompatible rewrite that embraces this: it drops the module abstraction, requires a
recent Neovim, and does little more than manage parsers and queries[^3].

The defining tension is therefore identity. For years "install nvim-treesitter" was
synonymous with "turn on Tree-sitter," and most users never distinguished the plugin
from the core feature. As core caught up, the plugin's scope shrank on purpose — good
for the ecosystem, confusing for users who inherited five-year-old config snippets that
no longer match either branch. **The GitHub repository is marked archived as of
mid-2026**, so the canonical source is read-only; the `master` branch remains available,
locked, for backward compatibility with Neovim 0.11 and earlier[^3].

## Getting Started

Install with any plugin manager and rebuild parsers on update. With lazy.nvim:

```lua
{
  'nvim-treesitter/nvim-treesitter',
  lazy = false,          -- this plugin does not support lazy-loading
  build = ':TSUpdate',   -- recompile parsers when the plugin updates
}
```

Requirements for the `main` branch are strict: Neovim 0.12+ (nightly), `tar`, `curl`,
the `tree-sitter-cli` (0.26.1+, installed via a package manager — **not** npm), and a C
compiler on `PATH`[^3]. Install parsers and enable highlighting for a filetype:

```lua
require('nvim-treesitter').install { 'rust', 'lua', 'python' }

vim.api.nvim_create_autocmd('FileType', {
  pattern = { 'rust', 'lua', 'python' },
  callback = function() vim.treesitter.start() end,   -- highlighting lives in core now
})
```

On the `master` branch the older module-based `setup { highlight = { enable = true } }`
form still applies; the two branches are not config-compatible.

## Architecture / How It Works

A parser is a C library compiled from a Tree-sitter grammar (`grammar.js` →
`parser.c` → shared object). nvim-treesitter's parser table maps each language to a
source repository and a **pinned revision**; `:TSInstall` / `:TSUpdate` clone at that
revision, run the CLI to generate `parser.c` when the repo doesn't ship it, invoke the C
compiler, and drop the result into a `parser/` directory on `runtimepath`[^2]. Pinning
matters because a parser's ABI and its query files must agree — a mismatched parser
against newer queries throws runtime errors, which is why the README insists on running
`:TSUpdate` whenever the plugin itself moves.

Queries are S-expression files (`highlights.scm`, `injections.scm`, `folds.scm`,
`indents.scm`, `locals.scm`) that pattern-match tree nodes and assign capture names.
Neovim's core reads them:

- **Highlighting** — `vim.treesitter.start()` walks `highlights.scm` captures and maps
  them to highlight groups. This is a core feature; the plugin only supplies the queries.
- **Folds / indentation** — `foldexpr` and `indentexpr` provided by core (folds) and by
  the plugin (indentation, still marked experimental).
- **Injections** — embedded languages (SQL in a string, Lua in a Vimscript heredoc)
  resolved via `injections.scm`; no per-language setup needed.
- **Locals** — scope/definition queries, retained only for limited backward compatibility
  and unused by the rewrite.

The coupling story is the whole point: parser ABI ⇄ query files ⇄ Neovim's Tree-sitter
runtime version all move together. The plugin's job is to keep those three in a
consistent set. Custom or unlisted grammars are added through a `User TSUpdate`
autocommand that registers an `install_info` entry (URL, revision, optional
`generate`/`location`/`queries`)[^3].

## Production Notes

**Two incompatible branches.** This is the dominant operational fact. `master` is the
classic, module-based plugin locked for Neovim 0.11 and earlier; `main` is the rewrite
targeting 0.12+ nightly. Config, commands, and mental model differ. Pin the branch
explicitly in your plugin spec and do not mix tutorials across the two.

**`:TSUpdate` is not optional.** Parsers are pinned; upgrading the plugin without
recompiling parsers leaves query files ahead of parser ABI and produces cryptic
highlight errors. Automate `build = ':TSUpdate'`. In headless/bootstrap scripts,
`install(...)` runs asynchronously — you must `:wait()` on it or the script exits before
compilation finishes[^3].

**Compiler dependency and Windows pain.** Every parser is compiled locally, so a working
C toolchain is mandatory. On Windows this is the most common failure mode (MSVC vs.
MinGW vs. zig cc mismatches); on locked-down machines without a compiler the plugin
cannot install anything.

**Large-file cost.** Tree-sitter highlighting re-parses on edit. On very large or
minified files the parse can dominate typing latency; Neovim disables Tree-sitter
highlighting past a size threshold, and users still hit slowdowns on multi-megabyte
buffers. Regex `:syntax` sometimes remains the pragmatic fallback there.

**No lazy-loading.** The plugin explicitly does not support being lazy-loaded; specs that
defer it (a common habit with other plugins) break parser discovery. Load it eagerly.

**Companion plugins version-track it.** `nvim-treesitter-textobjects` and
`nvim-treesitter-context` depend on the same runtime assumptions; the rewrite forces
matching branches across the family, so upgrade them together.

## When to Use / When Not

**Use when:**
- You run Neovim and want accurate, multi-language highlighting/folding/injection driven
  by real syntax trees rather than regex.
- You need embedded-language awareness (injections) or Tree-sitter-based text objects.
- You have a C compiler and can keep parsers updated with `:TSUpdate`.

**Avoid when:**
- You cannot provide a C toolchain (restricted machines) — you get nothing installable.
- You are on an old, pinned Neovim and want the newest features: you are stuck on the
  locked `master` branch.
- You edit huge/generated files where re-parsing latency outweighs the highlighting gain.
- You are not using Neovim at all — this plugin is Neovim-specific; the underlying library
  is what you want elsewhere.

## Alternatives

- tree-sitter/tree-sitter — the parsing library itself; use directly when building tooling
  or another editor's integration rather than configuring Neovim.
- neovim/neovim — with the rewrite, core's `vim.treesitter` API does more every release;
  use it directly when you only need highlighting and want zero plugin surface.
- nvim-treesitter/nvim-treesitter-textobjects — companion, not replacement; add it when you
  want syntax-aware text objects and motions on top of the parsers.
- sheerun/vim-polyglot — regex-syntax language bundle; use when you can't compile parsers
  or want a zero-dependency highlighting baseline.
- helix-editor/helix — a separate editor with Tree-sitter built in; use when you'd rather
  adopt an editor that ships this stack instead of assembling it in Neovim.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-04-18 | Repository created; module-based plugin over Neovim's experimental Tree-sitter runtime[^4]. |
| — | 2021 | Neovim 0.5 ships the Tree-sitter runtime; plugin becomes the de facto way to enable it[^5]. |
| — | 2023–2024 | Core absorbs highlight/fold/inspect surface (`:InspectTree` supersedes the playground plugin). |
| main rewrite | 2024–2026 | Incompatible `main` branch: drops modules, requires Neovim 0.12+, parser/query management only[^3]. |
| archived | 2026 | Repository marked archived (read-only); `master` locked for 0.11 backward compatibility. |

## References

[^1]: Max Brunsfeld et al., "Tree-sitter — a parser generator tool and incremental parsing library." https://tree-sitter.github.io/tree-sitter/
[^2]: nvim-treesitter README — parser installation and supported features. https://github.com/nvim-treesitter/nvim-treesitter/blob/main/README.md
[^3]: nvim-treesitter README (`main` branch) — rewrite caution, requirements, `install`/`TSUpdate`, custom parsers. https://github.com/nvim-treesitter/nvim-treesitter/blob/main/README.md
[^4]: GitHub API repository metadata (created_at 2020-04-18, license Apache-2.0). https://api.github.com/repos/nvim-treesitter/nvim-treesitter
[^5]: Neovim Tree-sitter documentation (`:h treesitter`). https://neovim.io/doc/user/treesitter.html

## Tags

neovim, tree-sitter, syntax-highlighting, lua, parser, editor-plugin, vim, code-editor, incremental-parsing, developer-tools
