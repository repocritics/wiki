# jesseduffield/lazygit

> A terminal UI for git that wraps the git CLI rather than reimplementing it — keyboard-driven staging, rebasing, and branch management over your existing repo.

[GitHub repo](https://github.com/jesseduffield/lazygit) ·
[License: MIT](https://github.com/jesseduffield/lazygit/blob/master/LICENSE)

## Overview

lazygit is a full-screen terminal UI (TUI) for git, written in Go and first released by Jesse Duffield in 2018[^1]. It presents the working tree, staged/unstaged hunks, branches, commits, stashes, and reflog as focusable panels, and binds common git operations — staging individual lines, interactive rebase, cherry-pick, bisect, amend, force-push — to single keystrokes. At ~80k stars it is among the most-starred Git interfaces on GitHub and the de facto reference for "git TUI."

The defining design decision is that lazygit **shells out to the system `git` binary** and parses its output, rather than linking a git library (libgit2, go-git). This is both its core strength and its main constraint: lazygit inherits your existing git config, hooks, aliases, credential helpers, and pager, and behaves like the git you already run — but it is coupled to git's command-line output format and requires a reasonably recent git to parse it reliably.

lazygit is aimed at developers who are comfortable with git concepts but tired of the ergonomics of the raw CLI — editing a rebase todo file by hand, stepping through `git add -p` hunks, or reconstructing state after a bad merge. It is not a git tutorial or an abstraction that hides git; it assumes you know what a rebase is and makes it faster to perform.

## Getting Started

```bash
# macOS / Linux (Homebrew)
brew install lazygit

# Go toolchain
go install github.com/jesseduffield/lazygit@latest

# Arch
pacman -S lazygit
```

Run it inside any git repository:

```bash
cd my-repo
lazygit
```

Common first keystrokes: `<space>` stages the selected file or hunk, `c` commits, `P` pushes, `p` pulls, `s` stashes, and `?` opens the context-sensitive keybinding menu. Configuration lives in a per-platform config directory (`~/.config/lazygit/config.yml` on Linux/macOS):

```yaml
# config.yml
gui:
  showFileTree: true
git:
  paging:
    colorArg: always
    pager: delta --dark --paging=never   # pipe diffs through delta
customCommands:
  - key: "<c-r>"
    command: "git rebase -i {{.SelectedLocalCommit.Sha}}^"
    context: "commits"
```

## Architecture / How It Works

lazygit is a Go binary built around a small set of coupled pieces:

- **TUI layer** — rendering and input use `jesseduffield/gocui`, the author's fork of the `awesome-gocui/gocui` terminal-UI library. Panels are gocui views; the app is an event loop that redraws on focus changes and git-state refreshes rather than continuously.
- **Command layer** — every git action constructs an argument vector and runs the real `git` executable via `os/exec`. Output is parsed (status porcelain, `git log` with format strings, diff hunks) back into the model. There is no in-process git implementation.
- **Interactive-rebase interception** — for operations that would normally drop you into `$EDITOR` (reordering, squashing, editing a rebase todo), lazygit sets `GIT_SEQUENCE_EDITOR` (and related env) to point back at itself, so it can rewrite the todo list programmatically without opening a text editor. This is how one-key "squash down" / "move commit up" works.
- **Undo** — implemented on top of git's reflog: lazygit records the reflog state and reverses operations by resetting to prior entries. It is not a private transaction log, so only reflog-visible operations are reliably undoable.

Because the model is "drive the CLI," lazygit's correctness tracks git's output stability. The project maintains an unusually thorough **integration test suite** that scripts real git repositories and replays recorded UI sessions against them, which is the practical guard against parser drift across git versions.

## Production Notes

**git version coupling.** lazygit parses porcelain output and depends on a modern git; very old system gits (common on stale LTS servers) can produce parse errors or missing features. Keep git current, and be aware that locale/`LANG` settings that translate git's output can break assumptions — running under a C/UTF-8 locale is the safe default.

**Large repositories.** Every refresh runs `git status`, `git log`, and related commands as subprocesses. On very large repos or slow filesystems (network mounts, huge histories) the UI can feel laggy on refresh, since cost is dominated by git itself, not lazygit. Rust alternatives that link libgit2 (see below) start faster on such repos.

**Destructive operations are one keypress away.** The ergonomics that make lazygit fast also make mistakes fast: dropping a commit, nuking the working tree, hard-resetting, or force-pushing are single keys. Reflog-based undo (`z`) recovers most *committed* mistakes, but discarding unstaged/untracked changes is not reflog-visible and is genuinely unrecoverable. Learn which actions prompt for confirmation and which do not.

**Hooks, signing, and credential helpers work — because it is real git.** Commit hooks, GPG/SSH commit signing, and credential managers all run exactly as they would on the command line. This is a feature, but it also means a slow pre-commit hook or a blocking credential prompt will stall the TUI; interactive prompts from subprocesses are surfaced but can be confusing inside a full-screen UI.

**Pager / diff integration.** lazygit renders diffs by shelling to git; piping through `delta` or another pager via `git.paging` gives syntax-highlighted diffs but can interact badly with paging modes — set `paging: never` on the pager so it does not try to page inside a panel.

**Config and keybindings are versioned surface.** `config.yml` keys and custom-command templating occasionally change between releases; custom commands referencing internal template fields (`.SelectedLocalCommit`, etc.) should be re-checked on upgrade. lazygit remains pre-1.0 (0.x), and while breaking config changes are rare, the project does not promise API-level stability.

## When to Use / When Not

**Use when:**
- You already know git and want to perform staging, rebasing, and history surgery faster than the raw CLI allows.
- You want a tool that respects your existing git config, hooks, aliases, and credential setup.
- You work over SSH / in a terminal and don't want a desktop GUI.
- You do a lot of partial staging (line/hunk level) or interactive rebase and find `git add -p` and todo-file editing tedious.

**Avoid when:**
- You want git *taught* to you — lazygit assumes you understand the underlying operations.
- You operate on very large monorepos where subprocess-per-refresh latency matters and a libgit2-backed tool would be snappier.
- You need a scriptable, non-interactive interface for CI or automation — that is the plain git CLI's job, not a TUI's.
- You live in an editor with a first-class git porcelain (Emacs + Magit) and don't want a second one.

## Alternatives

- extrawurst/gitui — Rust TUI backed by libgit2/git2-rs; faster startup and refresh on large repos. Use it when raw performance on big histories matters more than lazygit's breadth of one-key operations.
- jonas/tig — ncurses "text-mode interface for git," log/blame/diff browsing focused. Use it when you mainly want to *read* history and blame, not perform rebases and staging.
- magit/magit — the Emacs git porcelain, widely considered the most complete git UI. Use it when you already live in Emacs.
- jesseduffield/lazydocker — same author's TUI for Docker/compose; not a git alternative but the sibling project if you liked lazygit's model.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-05 | First public release; git TUI built on gocui[^1]. |
| 0.x series | 2018–present | Added interactive rebase UI, custom commands, bisect, and configurable keybindings incrementally. |
| worktrees | 2023 | Git worktree management added as a first-class panel. |
| ongoing | 2026 | Still 0.x (pre-1.0); actively maintained, frequent tagged releases[^2]. |

*Exact per-release dates are omitted where not independently verified; see the GitHub releases page[^2] for the authoritative changelog.*

## References

[^1]: jesseduffield/lazygit repository, created 2018-05-19. https://github.com/jesseduffield/lazygit
[^2]: lazygit releases and changelog. https://github.com/jesseduffield/lazygit/releases

## Tags

go, git, terminal, tui, cli, developer-tools, version-control, git-client, keyboard-driven
