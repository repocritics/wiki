# Canop/broot

> A terminal file navigator that shows a whole directory tree in one screen by pruning what won't fit, and can `cd` your shell to wherever you land.

[GitHub repo](https://github.com/Canop/broot) ·
[Official website](https://dystroy.org/broot/) ·
[License: MIT](https://github.com/Canop/broot/blob/main/LICENSE)

## Overview

broot (pronounced "b-root", invoked as `br`) is a TUI file manager written in Rust by Denys Séguret (Canop), first released in early 2019[^1]. Its defining idea is *balanced BFS descent*: instead of dumping a full recursive listing like `tree`, broot does a breadth-first walk and keeps only enough entries to fill the terminal, collapsing the rest into an "unlisted" count per directory[^2]. The result is a single navigable view of a large tree, filtered incrementally as you type.

The tool is aimed at people who live in the terminal and want `tree`, `cd`, `ls -la`, fuzzy find, and light file operations (mv/cp/rm/mkdir/chmod) in one interactive surface, without leaving the keyboard. It is `.gitignore`-aware by default, supports fuzzy/regex/content search, multiple side-by-side panels, a file preview panel, and a configurable verb system for launching external commands on the selection.

The central tension is scope. broot is deliberately a *navigator*, not a full orthodox file manager like Midnight Commander or ranger — its power comes from the search-driven tree view and the shell-integration trick, not from a large built-in feature set. Much of its behavior lives in user configuration (verbs, keybindings, skins), so the out-of-box experience is modest and the real capability is unlocked by investing in a config file.

## Getting Started

The binary alone cannot change the parent shell's directory, so broot installs a shell function named `br` that wraps it; the `--install` step writes that function into your shell rc files[^3].

```bash
# via cargo
cargo install broot
broot --install        # writes the `br` shell function, then restart the shell

# or via a package manager
brew install broot      # macOS / Linuxbrew
apt install broot       # Debian/Ubuntu (also in many other repos)
```

```bash
br              # launch; type letters to fuzzy-filter the tree
                #   /pat   regex search on names
                #   c/pat  search file contents
                #   alt-h  toggle hidden files, alt-i toggle gitignored
                # alt-enter on a directory: quit broot and cd the shell there
br -sdp ~       # show sizes, dates, permissions — an `ls -la` replacement
```

## Architecture / How It Works

broot is a single Rust binary built on the author's own terminal stack: `crossterm` for cross-platform terminal I/O and `termimad`/`minimad` for markdown-styled rendering, all maintained by Canop himself[^4]. This means the rendering and input layers are first-party, which keeps behavior consistent but concentrates maintenance on one person.

The tree engine is the core. A background walk computes the directory structure and, separately, expensive metadata (recursive sizes, dates, file counts) so navigation never blocks on `du`-style aggregation — sizes stream in after the tree is already interactive[^2]. Each keystroke cancels the in-flight search and starts a new one, which is why filtering feels immediate on large trees. The "unlisted" counts are a direct artifact of the balanced-descent algorithm choosing which nodes to render.

Interaction is driven by *verbs*. Built-in verbs (`:e` edit, `:rm`, `:cp`, `:mv`, `:mkdir`, `:gs` git-status view) and user-defined ones are bound to keys or invoked by typing `:name`. A verb can run an internal action or shell out to an external command with placeholders like `{file}`, and can be configured to leave broot, stay, or open a new panel. Configuration lives in a `conf.toml`/`conf.hjson` file (Hjson and JSON also accepted) covering verbs, keybindings, skins/colors, and default flags.

Shell integration is a deliberate two-part design: the compiled program writes a command (e.g. a `cd`) to a file descriptor, and the `br` shell function reads and `eval`s it after broot exits. This is the only way a child process can affect the parent shell's working directory, and it is why `cd`-on-quit requires the installed function rather than the bare binary.

## Production Notes

- **`br` vs `broot`.** Running the raw `broot` binary works for browsing but silently loses the `cd`-on-exit feature — the single most common "why doesn't this work" issue. You must run `--install` and use `br`. In restricted or non-interactive shells where the function isn't sourced, the feature is unavailable.
- **Shell coverage.** The `br` function ships for bash, zsh, and fish; other shells (nushell, elvish, xonsh) need manual wiring and may lag. Check the install docs for your specific shell rather than assuming.
- **Preview and images.** High-resolution image preview only works in terminals implementing Kitty's graphics protocol (Kitty, WezTerm); elsewhere previews degrade to lower-fidelity or text[^5]. Large-file and binary preview is intentionally limited.
- **Large trees and network mounts.** Recursive size/date computation walks the whole subtree in the background. On very large directories, slow disks, or networked/`sshfs` mounts this background work can be heavy even though the UI stays responsive; `--whale-spotting` (`-w`) leans into this and will do more scanning.
- **Config is a moving target across a fast minor cadence.** broot ships frequent 1.x minor releases; verb syntax, config keys, and defaults have accreted over dozens of versions. A config written years ago mostly keeps working, but new features (search prefixes, panel verbs, staging area) are only discoverable by reading the changelog or docs, not from the UI.
- **`.gitignore` behavior can surprise.** Because gitignored and hidden files are filtered by default, files you expect to see may be absent until you toggle `alt-i`/`alt-h`. This is correct but trips up first-time users doing cleanup.
- **Bus-factor.** The project and its whole dependency substack are effectively one maintainer. It is healthy and actively released, but that is a concentration risk for teams standardizing on it.

## When to Use / When Not

**Use when:**
- You want a fast, keyboard-driven way to *see* and move through big directory trees and jump the shell into them.
- You want fuzzy/regex/content search fused with a live tree view, respecting `.gitignore`.
- You want a lightweight `tree`/`ls`/`cd`/`du`-lite replacement without adopting a heavyweight file manager.
- You'll invest a little in a config file for custom verbs and keybindings.

**Avoid when:**
- You need a full orthodox file manager with rich built-in operations, plugins, and archive/mount handling (ranger, Midnight Commander, yazi fit better).
- You only ever want a fuzzy filename or content finder (fzf, fd, ripgrep are simpler and composable).
- You work primarily in shells or terminals without the `br` function or Kitty graphics, and the `cd`/preview features are your reason for adopting it.

## Alternatives

- sxyazi/yazi — async Rust TUI file manager with a broader feature set and image preview; pick it when you want a full file manager, not just a navigator.
- sharkdp/fd — fast, `.gitignore`-aware file *finder*; use it when you want scriptable path search rather than an interactive tree.
- BurntSushi/ripgrep — content search only; use for grepping code at speed, not navigation.
- gokcehan/lf / ranger — orthodox terminal file managers; use when you want a classic two-pane/miller-columns manager with heavy customization.
- junegunn/fzf — general fuzzy picker; use to fuzzy-select from any list (including piped file lists) rather than browse a tree.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.4.1 | 2019-01-07 | Earliest public releases on GitHub[^6]. |
| v0.7.0 | 2019-03-07 | Early feature growth (verbs, search modes). |
| v1.0.0 | 2020-09-01 | 1.0 milestone; multi-panel and preview era[^6]. |
| v1.6.0 | 2021-06-16 | Staging area for multi-file operations. |
| v1.14.0 | 2022-07-05 | Continued search/preview refinements. |
| v1.21.0 | 2023-03-17 | Ongoing verb and config expansion. |
| v1.44.0 | 2024-09-07 | Mature 1.x line. |
| v1.58.0 | 2026-07-10 | Latest release at time of writing[^6]. |

## References

[^1]: broot repository, Canop/broot. https://github.com/Canop/broot
[^2]: broot documentation — overview and navigation. https://dystroy.org/broot/
[^3]: broot installation instructions (the `br` shell function and `--install`). https://dystroy.org/broot/install/
[^4]: crossterm and termimad, both authored by Denys Séguret (Canop). https://github.com/Canop/termimad
[^5]: broot documentation — preview panel and Kitty graphics support. https://dystroy.org/broot/preview/
[^6]: broot GitHub releases (dates verified via GitHub API). https://github.com/Canop/broot/releases

## Tags

rust, cli, tui, file-manager, terminal, directory-tree, fuzzy-search, gitignore, developer-tools, cross-platform
