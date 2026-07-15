# ajeetdsouza/zoxide

> A smarter `cd`: a Rust reimplementation of `z`/autojump that ranks visited directories by frecency and jumps to them from any major shell.

[GitHub repo](https://github.com/ajeetdsouza/zoxide) ·
[crates.io](https://crates.io/crates/zoxide) ·
[License: MIT](https://github.com/ajeetdsouza/zoxide/blob/main/LICENSE)

## Overview

zoxide is a command-line tool that remembers the directories you visit and lets
you jump to them with a partial name. `z proj` cds into the highest-ranked
directory whose path matches `proj`. It is a compiled Rust binary in the lineage
of `rupa/z` (a zsh/bash script) and `wting/autojump` (a Python daemon), and it
exists mainly because those predecessors were shell-specific, slow to start, or
carried a runtime dependency. zoxide ships a single static binary with no runtime
requirements (fzf is optional, for interactive selection) and supports bash, zsh,
fish, PowerShell, Nushell, elvish, xonsh, and any POSIX shell.

The defining design choice is *frecency* ranking — a frequency-times-recency
score borrowed from Mozilla's URL bar[^1]. Each directory accumulates a score on
visits; scores decay over time so recently-and-often used paths win. The tradeoff
this creates: zoxide can only jump to directories it has *already seen*. A fresh
install is empty and behaves like a broken `cd` until you have browsed around or
seeded it with `zoxide import`. It is a navigation accelerator for interactive
shells, not a general directory resolver, and treating it as the latter is the
most common source of "it doesn't work" confusion.

It is widely packaged (Homebrew, most Linux distros, winget, cargo) and actively
maintained — the repository has ~38k stars and commits land regularly[^2].

## Getting Started

```sh
# install (any platform with Rust; see README for OS package managers)
cargo install zoxide --locked
# or: brew install zoxide  /  winget install ajeetdsouza.zoxide
```

Add the init hook to the **end** of your shell config, then restart the shell:

```sh
# ~/.bashrc / ~/.zshrc
eval "$(zoxide init bash)"     # or: zoxide init zsh

# ~/.config/fish/config.fish
zoxide init fish | source
```

```sh
z foo            # jump to the top-ranked dir matching "foo"
z foo bar        # dir matching both "foo" and "bar", in order
z ~/src          # plain paths still work like cd
zi foo           # interactive pick among matches (requires fzf)
zoxide import z  # seed the database from an existing z/autojump/fasd install
```

## Architecture / How It Works

zoxide is not a shell plugin in the traditional sense. `zoxide init <shell>`
prints a block of shell script to stdout; the `eval`/`source` in your rc file
installs three things: the `z` / `zi` wrapper functions, and a hook that fires on
directory change (`pwd`, the default) or on every prompt (`prompt`). The hook
calls `zoxide add "$PWD"`, which bumps that directory's score in the database.
The `z` function shells out to `zoxide query` and cds to whatever it returns. All
the intelligence lives in the binary; the shell code is a thin, generated shim.

The database is a single binary file (`db.zo`) under `$_ZO_DATA_DIR` — a
platform-specific data directory (`~/.local/share` on Linux, Application Support
on macOS, `%LOCALAPPDATA%` on Windows). It stores `(path, score, last-access)`
rows. There is no daemon and no background process; every operation is a
short-lived binary invocation reading and rewriting this file.

Matching is ordered keyword matching, not fuzzy search. Query keywords must appear
in the path in the given order, and the **last** keyword must match a segment of
the path's final component. `z src zox` matches `~/src/.../zoxide` but not
`~/zoxide/src`. This ordering rule is why zoxide feels precise rather than
scattershot, and why users sometimes expect a match that the algorithm
deliberately rejects[^3].

Scores decay via an *aging* algorithm[^4]: when the sum of all scores exceeds
`_ZO_MAXAGE` (default 10000) every entry is multiplied down, and entries that fall
below a floor are evicted. This bounds the database size and lets stale
directories age out without manual pruning. Interactive selection (`zi`, or
`z foo<Space><Tab>` completions) delegates to fzf; zoxide itself contains no TUI.

## Production Notes

- **Empty-database cold start.** Right after install, `z anything` fails because
  nothing has been recorded. This is expected. Seed with `zoxide import z` /
  `autojump` / `fasd`, or just use the shell normally for a while. Many bug
  reports are actually this.
- **It only jumps to visited directories.** zoxide cannot cd into a path you have
  never opened. Plain paths (`z ./relative`, `z ~/abs`) still pass through to a
  cd-like fallback, but the "smart" part is limited to the database.
- **Do not sync `db.zo` across machines with different layouts.** The database is
  absolute-path keyed. Copying it between a laptop and a server whose home dirs
  or project roots differ produces matches that resolve to nonexistent paths.
- **Symlinks are not resolved by default.** Two symlinked views of the same tree
  get separate entries. Set `_ZO_RESOLVE_SYMLINKS=1` if you want canonical paths.
- **Init ordering matters.** The eval line must go at the *end* of the rc file.
  On zsh it must come *after* `compinit`, or `<Space><Tab>` completions silently
  fail (you may need `rm ~/.zcompdump*; compinit`). Completions need bash 4.4+.
- **fzf version floor.** `zi` and interactive completions require fzf ≥ 0.51.0;
  older fzf (still shipped by some LTS distros) breaks interactive mode with
  opaque errors. Nushell (≥ 0.89) and elvish (≥ 0.18) also have minimum versions.
- **`--cmd cd` replaces cd in interactive shells only.** Scripts invoking the
  real `cd` builtin are unaffected, but the muscle-memory change (and the fact
  that `cd nonexistent` now behaves differently) surprises some users; try it
  before committing it to a shared dotfiles repo.
- **Distro packages lag.** Debian/Ubuntu builds are often many versions behind
  and the README explicitly deprecates `apt install zoxide` in favor of the
  install script or cargo.

## When to Use / When Not

**Use when:**
- You switch between many directories interactively and want fewer keystrokes.
- You use several shells and want one tool with consistent behavior across them.
- You want a fast, dependency-free native binary rather than a Python/shell script.
- You are migrating off z, autojump, or fasd and want to keep your history.

**Avoid when:**
- You need deterministic paths in scripts — use plain `cd`; zoxide's ranking is
  interactive-shell state, not a scriptable resolver.
- You want to jump to directories you have never visited (that is `find`/`fd`).
- You cannot modify the shell rc file (locked-down or ephemeral environments).
- You are actually looking for command *history* search — that is atuin/mcfly, a
  different problem than directory navigation.

## Alternatives

- rupa/z — the original POSIX-shell `z`; use it when you want a zero-binary,
  pure-script tool on bash/zsh and don't mind slower startup.
- wting/autojump — Python-based ancestor; reasonable if it's already installed
  and you accept a Python runtime dependency.
- skywind3000/z.lua — a Lua reimplementation with more flags and enhanced
  matching; use it when you have Lua available and want extra tuning knobs.
- agkozak/zsh-z — a pure-zsh rewrite of z; choose it if you are zsh-only and
  prefer no external binary at all.
- junegunn/fzf — complementary, not a replacement: fzf is the fuzzy picker zoxide
  calls for `zi`. Use fzf's own `Alt-C`/`**` widgets if you want fuzzy dir
  selection without frecency ranking.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-03 | Repository created; Rust reimplementation of `z`/autojump[^2]. |
| 0.x | 2020–2021 | Binary `db.zo` format, `zoxide import`, broad shell coverage (fish, PowerShell, elvish, xonsh) added. |
| 0.x | 2022–2024 | Nushell support, fzf-based `zi` interactive selection, aging algorithm refinements. |
| current | 2026-07 | Still 0.x, actively maintained, ~38k stars, MIT-licensed[^2]. |

## References

[^1]: "Frecency" — combined frequency/recency scoring, originating in Mozilla's Places/awesomebar. https://en.wikipedia.org/wiki/Frecency
[^2]: zoxide repository and release history. https://github.com/ajeetdsouza/zoxide
[^3]: zoxide matching algorithm (keyword order; last keyword matches final path component). https://github.com/ajeetdsouza/zoxide/wiki/Algorithm#matching
[^4]: zoxide aging algorithm (`_ZO_MAXAGE`, score decay and eviction). https://github.com/ajeetdsouza/zoxide/wiki/Algorithm#aging

## Tags

rust, cli, shell, terminal, directory-navigation, cd-replacement, frecency, developer-tools, productivity, fzf, cross-platform
