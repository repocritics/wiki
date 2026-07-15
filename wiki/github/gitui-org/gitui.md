# gitui-org/gitui

> Terminal UI for git, written in Rust, built on libgit2 rather than shelling out to the git binary.

[GitHub repo](https://github.com/gitui-org/gitui) ·
[crates.io](https://crates.io/crates/gitui) ·
[License: MIT](https://github.com/gitui-org/gitui/blob/master/LICENSE.md)

## Overview

GitUI is a keyboard-driven terminal front-end for git that aims to give you the
staging, diffing, stashing, and log-browsing comfort of a git GUI without
leaving the terminal. It was started by Stephan Dilly (`extrawurst`) in 2020 and
has since moved from his personal namespace to the `gitui-org`
organization[^1]. The project is still pre-1.0 and self-described as beta,
though it is stable enough that its authors use it to develop itself.

The defining design choice is that GitUI talks to repositories through
**libgit2** (via the `git2` Rust bindings), not by parsing the output of the
`git` CLI. This is what lets it stay responsive on very large repositories —
the maintainer's motivating complaint was that popular git GUIs freeze or fall
over on giant repos. A README benchmark parsing the ~900k-commit Linux kernel
history clocks GitUI at 24s / 0.17 GB versus lazygit at 57s / 2.6 GB and tig at
4m20s[^2]. Treat those numbers as author-run and directional, not independently
audited.

The tradeoff of the libgit2 route is scope: libgit2 does not implement every
git feature, so GitUI inherits gaps. There is no git-lfs support, no sparse
checkout support, and `credential.helper` for HTTPS remotes must be configured
explicitly[^3]. GitUI is explicitly positioned as a companion to the git shell,
not a replacement for it — it deliberately optimizes the operations that are
painful on the command line (staging hunks/lines, stashing, blame, log search)
and leaves the rest to `git`.

## Getting Started

```sh
# From crates.io (needs Rust/cargo >= 1.88, a C compiler, and perl for openssl)
cargo install gitui --locked

# Or via a package manager
brew install gitui          # macOS
pacman -S gitui             # Arch
winget install gitui        # Windows
```

```sh
# Run inside any git repository
cd my-repo
gitui

# Common keys: 1-5 switch tabs, [space] stage/unstage, [enter] open,
# c commit, s stash, ? context help. Run with logging:
gitui -l
```

Prebuilt release binaries (statically linked musl on Linux, arm64/x86 on macOS,
`.msi` and tarball on Windows) are published on the releases page for users who
do not want to compile.

## Architecture / How It Works

GitUI is a Cargo workspace split into a few crates. The important separation is
between the UI binary and **`asyncgit`**, the crate that wraps `libgit2` and
runs potentially slow git operations off the render thread. Expensive work
(diffing, log walking, status) is dispatched to a background thread pool; when
it finishes it posts an event back to the UI, which re-renders. This
asynchronous git layer is the reason the interface stays interactive while a
large status or log computation is in flight — the event loop is never blocked
on a synchronous libgit2 call.

The terminal rendering is built on **ratatui**, the community fork of the
now-archived `tui-rs` crate; GitUI migrated to ratatui after the original was
abandoned[^4]. Input is entirely keyboard-driven by default, with a context
help overlay so users are not expected to memorize bindings; the keymap and
color theme are both customizable via config files (`KEY_CONFIG.md`,
`THEMES.md`).

Two consequences follow from the libgit2 choice. First, the build pulls in a C
toolchain and openssl — hence the perl and C-compiler build requirements, and
the historical openssl friction on Windows. Second, behavior tracks what
libgit2 supports rather than what your installed `git` supports; features that
live only in the git CLI (or in newer git plumbing) are simply absent until
libgit2 gains them or GitUI implements them itself. The crate forbids `unsafe`
code project-wide, advertised via the `unsafe-forbidden` badge[^5].

## Production Notes

- **HTTPS credentials need explicit setup.** GitUI does not transparently reuse
  every OS credential helper flow; `credential.helper` must be configured, and
  users behind SSO/2FA HTTPS remotes are a recurring source of push/fetch
  failures (see issue #800). SSH key auth is generally the smoother path.
- **GPG commit signing is incomplete.** Signing works but with documented
  shortcomings (issue #97) — do not assume parity with `git commit -S`.
- **No git-lfs, no sparse checkout.** Repositories relying on LFS pointers or
  sparse-checkout will not behave correctly; this is a libgit2/scope
  limitation, not a bug (issues #2812, #1226).
- **No interactive rebase yet.** Interactive rebase is a stated road-to-1.0
  goal rather than a shipped feature; reach for the git shell for rebase
  surgery.
- **MSRV moves.** The minimum supported Rust version is 1.88 and has been
  bumped repeatedly over the project's life, which can break `cargo install`
  on older toolchains — pin/upgrade Rust before filing build issues.
- **Pre-1.0 stability contract.** Config file formats and keybindings have
  changed across releases; expect to revisit your theme/keymap on upgrades.
- **libgit2 build dependencies.** Because it links libgit2/openssl, the source
  build is heavier than a pure-Rust tool — CI and fresh machines need perl and
  a C compiler, not just cargo.

## When to Use / When Not

**Use when:**
- You live in the terminal but want a GUI-grade experience for staging hunks,
  lines, stashing, blame, and log search.
- You work in very large repositories where GUI clients become unresponsive.
- You want a single portable static binary with no runtime dependencies.

**Avoid when:**
- You depend on git-lfs, sparse checkout, or interactive rebase inside the tool.
- You need mouse-heavy interaction or a feature-maximal TUI — lazygit covers
  more git surface today.
- You want a read-only log/diff pager — tig is lighter and more focused.
- You are on a toolchain older than the current MSRV and cannot upgrade Rust.

## Alternatives

- jesseduffield/lazygit — use instead when you want the broadest feature set and
  mouse support; it shells out to `git` and covers interactive rebase and more.
- jonas/tig — use instead when you mainly need a fast, lightweight ncurses
  log/diff/blame browser rather than a full staging UI.
- git-up/GitUp — use instead when you are on macOS and want a graphical,
  map-view GUI rather than a terminal tool.
- magit/magit — use instead when you already work in Emacs and want the most
  complete keyboard git porcelain available.
- Wilfred/difftastic — complementary, not a replacement: a structural diff tool
  you can wire in when GitUI's line diffs are not enough.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-03-16 | Project started by extrawurst on GitHub[^1]. |
| 0.1.x | 2020 | First crates.io releases; libgit2-backed async core established. |
| pre-1.0 | 2023 | Migrated terminal rendering from tui-rs to ratatui[^4]. |
| ownership move | — | Repository transferred to the `gitui-org` organization[^1]. |
| current | 2026 | Still pre-1.0 beta; MSRV Rust 1.88; ~22k stars[^6]. |

Exact per-release dates are intentionally omitted where not verified; see the
releases page for the canonical changelog.

## References

[^1]: gitui repository, `gitui-org/gitui` (redirected from `extrawurst/gitui`). https://github.com/gitui-org/gitui
[^2]: README benchmark, RustBerlin meetup presentation (parsing the Linux kernel git history). https://github.com/gitui-org/gitui#3-benchmarks
[^3]: README, "Known Limitations". https://github.com/gitui-org/gitui#5-known-limitations
[^4]: ratatui, community fork of the archived tui-rs crate. https://github.com/ratatui/ratatui
[^5]: rust-secure-code "safety-dance" / `unsafe-forbidden` badge. https://github.com/rust-secure-code/safety-dance
[^6]: GitHub repository metadata (stars/forks/license), fetched 2026-07-15. https://github.com/gitui-org/gitui

## Tags

rust, git, terminal-ui, tui, cli, developer-tools, ratatui, libgit2, version-control, git-client
