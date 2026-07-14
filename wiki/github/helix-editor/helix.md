# helix-editor/helix

> A post-modern modal text editor — Kakoune's selection-first editing model, Rust internals, and LSP/tree-sitter batteries included out of the box.

[GitHub repo](https://github.com/helix-editor/helix) ·
[Official website](https://helix-editor.com) ·
[License: MPL-2.0](https://github.com/helix-editor/helix/blob/master/LICENSE)

## Overview

Helix is a terminal-based modal text editor written in Rust, started by Blaž Hrastnik (archseer) and first tagged in 2022[^1]. Its editing model is copied deliberately from Kakoune rather than Vim: the primitive is a *selection*, and commands act on the current selection. Where Vim composes verb-then-motion (`dw` = delete word), Helix composes selection-then-verb (`wd` = select word, then delete). Every action shows its effect on-screen before you commit it, which makes multiple selections — the model's other pillar — visible and predictable instead of an afterthought.

The project's defining stance is *batteries included, no plugins*. Language server support, Debug Adapter Protocol support, tree-sitter syntax highlighting and text objects, a fuzzy file picker, and global search all ship in the single binary with no configuration. There is no Vimscript/Lua/Emacs-Lisp extension layer in the stable releases; behavior is configured through two TOML files, not scripted. This is the central tradeoff around which every "should I switch" discussion turns: you trade the Neovim/Emacs plugin universe for a coherent, zero-setup default that works the moment it is installed.

Helix uses calendar versioning (`YY.MM`) rather than semver[^2], and is actively maintained — the repository has a large contributor base and frequent pushes, with tens of thousands of stars reflecting real adoption among developers who want a modal editor without maintaining a config repo.

## Getting Started

```bash
# macOS
brew install helix
# Arch
sudo pacman -S helix
# from source (requires a C compiler for tree-sitter grammars)
git clone https://github.com/helix-editor/helix
cd helix
cargo install --path helix-term --locked
```

Run `hx --health` first — it reports whether the runtime directory (grammars + queries) and language servers are found. The most common first-run problem is a missing runtime, which shows up as no syntax highlighting.

Configuration lives in `~/.config/helix/`:

```toml
# ~/.config/helix/config.toml
theme = "onedark"

[editor]
line-number = "relative"
bufferline = "multiple"

[editor.cursor-shape]
insert = "bar"

[keys.normal]
C-s = ":w"          # remap Ctrl-S to save
```

## Architecture / How It Works

Helix is a Cargo workspace split into focused crates rather than a monolith:

- **`helix-core`** — the editing engine: the rope (text is stored in a `ropey` rope, so edits are O(log n) and large files stay responsive), selections, transactions, tree-sitter integration, and text objects. All editing is expressed as immutable transactions applied to the document, which is also what makes undo and multiple-cursor edits consistent.
- **`helix-view`** — documents, views (splits), and editor state above the raw buffer.
- **`helix-term`** — the actual TUI application, keymaps, commands, and the event loop.
- **`helix-tui`** — the rendering layer, originally forked from `tui-rs` (the predecessor of ratatui).
- **`helix-lsp`** / **`helix-dap`** — language server and debug adapter clients.
- **`helix-loader`** — runtime discovery, grammar fetching and compilation.

Syntax highlighting, indentation, and text objects are all driven by **tree-sitter** grammars and `.scm` query files living in `runtime/queries/<lang>/`. Grammars are C libraries fetched and compiled with `hx --grammar fetch` / `hx --grammar build`; the compiled `.so`/`.dll` files plus the query files constitute the *runtime directory* that must be discoverable at launch (via install location or `HELIX_RUNTIME`).

The fuzzy picker is powered by **nucleo**, a matcher written by the Helix authors and now used elsewhere in the Rust ecosystem. Language configuration — which server to launch, formatters, file-type associations, indentation — is data in `languages.toml`, not code, so adding or overriding a language means editing TOML, not writing a plugin.

## Production Notes

**No stable plugin system.** This is the single most important operational fact. A scripting/plugin layer built on **Steel** (a Scheme dialect) has been in development for a long time but is not part of the stable releases and lives behind separate builds/branches[^3]. Until it lands, anything not covered by the built-ins cannot be added. Do not adopt Helix expecting to fill a gap with "a plugin for that" — evaluate whether the built-ins already cover your workflow.

**Runtime directory is the #1 install footgun.** Package-manager installs usually wire it up; manual `cargo install` and portable/AppImage setups frequently do not, and the symptom (a working editor with no highlighting and no completion) does not obviously point at the cause. Always run `hx --health` after install and set `HELIX_RUNTIME` if the runtime is not auto-detected.

**Limited built-in feature surface by design.** There is no file-tree sidebar (a `[editor.file-picker]` and buffer picker substitute for it), git integration is limited to gutter diff signs plus blame in newer builds rather than a full git UI, and there is no integrated terminal split. Teams migrating from a heavily-extended Neovim setup routinely hit one of these and need to change habits rather than reconfigure.

**Config and language changes can break across releases.** Because there is no compatibility layer and CalVer implies no semver promises, TOML keys and default keymaps occasionally change between releases; re-read the release notes on upgrade rather than assuming your config carries forward untouched.

**Grammar compilation needs a toolchain.** Building grammars from source requires a C compiler; minimal containers/CI images without one will fail grammar builds. Prebuilt-runtime distro packages avoid this.

**Macros exist but are minimal.** Recording (`Q`/`q`) and replay cover simple repetition; there is no scripting-grade automation, and complex multi-step edits are expected to be done with multiple selections instead of recorded macros.

## When to Use / When Not

**Use when:**
- You want modal editing with LSP, DAP, tree-sitter, fuzzy find, and global search working on day one with no config.
- You prefer Kakoune's selection-first model and want on-screen feedback for every action.
- You are tired of maintaining a Neovim/LazyVim config and want a stable default you don't babysit.
- You edit large files and want rope-backed responsiveness.

**Avoid when:**
- Your workflow depends on specific plugins (advanced git UIs, DBUIs, org-mode, REPL integrations) — the ecosystem does not exist yet.
- You need an integrated terminal, file-tree sidebar, or heavy IDE-like panels out of the box.
- You require a GUI (Helix is terminal-first; a native renderer is only aspirational).
- Your muscle memory is deeply Vim and you cannot afford to relearn the reversed verb order.

## Alternatives

- neovim/neovim — use instead when you need the plugin ecosystem, Lua scripting, and maximal extensibility.
- mawww/kakoune — use instead when you want the same selection model plus client-server architecture and shell-based extensibility.
- zed-industries/zed — use instead when you want a GUI, collaboration, and a Vim mode rather than a terminal modal editor.
- vim/vim — use instead when you need the most ubiquitous, everywhere-preinstalled modal editor with decades of plugins.
- emacs-mirror/emacs — use instead when you want a fully programmable environment and don't mind building it yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 22.03 | 2022-03 | First tagged public release; adopts CalVer[^1]. |
| 22.05 | 2022-05 | Early rapid-iteration releases. |
| 22.12 | 2022-12 | Sticky selections, expanded language support. |
| 23.10 | 2023-10 | Continued LSP/DAP and picker improvements. |
| 24.03 | 2024-03 | Ongoing language and UX refinements. |
| 24.07 | 2024-07 | Further releases on the CalVer cadence. |
| 25.01 | 2025-01 | Recent stable line[^2]. |

(Helix does not use semantic versioning; treat every release as potentially config-affecting.)

## References

[^1]: Helix repository and releases — helix-editor/helix. https://github.com/helix-editor/helix/releases
[^2]: Helix documentation (install, configuration, keymap). https://docs.helix-editor.com/
[^3]: Steel — an embedded Scheme used for the in-development Helix plugin system. https://github.com/mattwparas/steel

## Tags

rust, text-editor, modal-editor, terminal, tui, lsp, tree-sitter, kakoune, vim, developer-tools, cli, calver
