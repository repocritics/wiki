# dandavison/delta

> A syntax-highlighting pager that reformats git, diff, grep, and blame output — it styles diffs, it does not compute them.

[GitHub repo](https://github.com/dandavison/delta) ·
[Official website](https://dandavison.github.io/delta/) ·
[License: MIT](https://github.com/dandavison/delta/blob/main/LICENSE)

## Overview

Delta is a terminal pager written in Rust that sits between git and your
terminal, reparsing unified-diff output and re-emitting it with syntax
highlighting, word-level change highlighting, line numbers, and optional
side-by-side layout[^1]. The package is named `git-delta` in most package
managers, but the executable is `delta`. It is not a diff algorithm: git (or
`diff`, `grep`, `rg`) still produces the diff; delta only restyles the stream.

The core value is legibility. Standard `git diff` marks changes at line
granularity with red/green; delta adds language-aware syntax highlighting (via
the same theme and grammar assets used by `sharkdp/bat`), infers within-line
edits with a Levenshtein-based algorithm to highlight just the changed tokens,
and can render two-column side-by-side views with wrapping. It also improves
`git blame`, merge-conflict display, and grep output from `rg` / `git grep`.

The defining tension is configuration surface versus a clean default. Delta is
heavily customizable — 20-plus stylable elements, box/line decorations, named
feature presets, emulation modes for `diff-highlight` and `diff-so-fancy` — and
that flexibility is the draw, but the same surface makes it easy to end up with
a `.gitconfig` `[delta]` block nobody remembers the meaning of. It is a
polished, actively maintained tool (31k stars, last pushed within days as of
2026), carrying a large open-issue backlog (~400) typical of a popular,
option-rich CLI[^2].

## Getting Started

Install (Homebrew shown; also in apt, dnf, pacman, cargo, scoop, etc.[^3]):

```sh
brew install git-delta
```

Add to `~/.gitconfig`:

```gitconfig
[core]
    pager = delta
[interactive]
    diffFilter = delta --color-only
[delta]
    navigate = true      # n / N jump between diff sections
    side-by-side = true
    line-numbers = true
[merge]
    conflictStyle = zdiff3
```

Then `git diff`, `git show`, `git log -p`, and `git blame` route through delta
automatically. `delta --help` prints the full manual; `delta --show-syntax-themes`
previews color themes.

## Architecture / How It Works

Delta is a streaming line processor, not a differ. Its input is the ANSI or
plain text that git/diff/grep already emit; it detects structural markers
(`diff --git`, `@@` hunk headers, `+`/`-` lines, commit and file headers) and
rewrites each region. Because it consumes git's own output, in interactive
contexts like `git add -p` it must be invoked as `delta --color-only` through
`interactive.diffFilter` so it restyles rather than re-parses already-processed
content.

Syntax highlighting is provided by the `syntect` crate operating over Sublime
Text grammar and theme assets — the same asset set as bat, which is why every
bat theme is available in delta[^1]. Highlighting is applied per line within a
hunk after delta has classified the line as added/removed/context. Word-level
("intra-line") highlighting runs an edit-distance inference between the removed
and added versions of a changed line to mark the minimal changed span rather
than the whole line.

Layout is computed after styling: line numbers, side-by-side column splitting,
and line wrapping are presentation passes over the classified, highlighted
lines. Delta does not page on its own — by default it shells out to `less`
(honoring `PAGER`/`DELTA_PAGER`), passing flags for raw ANSI and, when
`navigate` is set, search patterns that implement the `n`/`N` section jumping.
`--hyperlinks` emits OSC 8 terminal escape sequences so commit hashes and file
paths become clickable, with hosting-provider URL templates for GitHub, GitLab,
SourceHut, and Codeberg.

## Production Notes

- **It renders diffs, it doesn't diff.** Delta cannot show semantic/structural
  changes; a reformatted line still appears as a full delete+add. For AST-aware
  diffing use difftastic instead — a different category of tool.
- **Paging quirks come from `less`.** Most "delta doesn't scroll / colors look
  wrong / output truncated" reports trace to the underlying pager and the `LESS`
  environment. Truecolor requires a 24-bit-capable terminal; on 256-color
  terminals themes degrade to the nearest palette. If output is being captured
  or piped, set `--paging=never`.
- **`--color-only` in interactive mode is mandatory.** Omitting it from
  `interactive.diffFilter` produces broken or doubly-escaped output in
  `git add -p` and similar flows, because delta would be reparsing colored input.
- **Large diffs cost CPU.** Syntax highlighting and intra-line inference are
  per-line work; very large diffs (generated files, vendored trees, lockfiles)
  are noticeably slower than raw git. Restrict with pathspecs or drop delta for
  those cases.
- **Copy-paste is a feature, not free.** By default delta strips leading
  `+`/`-` markers so code copies cleanly out of the diff — convenient, but it
  means what you see is not byte-identical to raw diff text.
- **Merge-conflict display wants `zdiff3`.** Delta's improved conflict rendering
  assumes git's `zdiff3` conflict style; with the older `merge` style the
  output is less useful.
- **Config drift.** Feature presets and the `[delta]` block accumulate. Prefer
  named `[delta "feature"]` groupings activated via `features =` so intent stays
  legible, and re-check with `delta --help` after upgrades since option names
  and defaults have shifted across releases.

## When to Use / When Not

**Use when:**
- You read diffs, blame, or `log -p` in a terminal daily and want syntax
  highlighting and clearer change markers.
- You want side-by-side review without leaving the CLI.
- You already use bat and want matching themes across your tooling.
- You want a drop-in that stays close to git's default semantics.

**Avoid when:**
- You need structural/semantic diffs (renamed variables, moved blocks) — use an
  AST differ.
- You run in constrained or non-truecolor terminals where the styling degrades
  and adds little.
- You want zero-config: delta's payoff comes with some `.gitconfig` investment.
- You are scripting/parsing diff output downstream — pipe raw git, not delta.

## Alternatives

- so-fancy/diff-so-fancy — use when you want a lighter, Perl-based prettifier
  and don't need syntax highlighting or side-by-side; delta can emulate it.
- Wilfred/difftastic — use when you want structural, syntax-aware diffs that
  actually understand the language, not just highlighting.
- sharkdp/bat — use for paging/printing files generally; it's the sibling that
  shares delta's highlighting assets but is not diff-specialized.
- walles/riff — use when you want a smaller Rust diff refiner focused on
  intra-line highlighting with less configuration.
- git contrib `diff-highlight` — use when you want the minimal, dependency-free
  option shipped with git itself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-06 | First public release; Rust pager over git diff output[^2]. |
| 0.4–0.6 | ~2020 | Side-by-side view and line-numbers added. |
| 0.7–0.9 | ~2021 | `git blame` styling; improved merge-conflict display. |
| 0.10–0.13 | ~2022 | Grep/ripgrep highlighting; `--hyperlinks` provider links; `diff-so-fancy` emulation. |
| 0.16–0.18 | 2023–2024 | Ongoing feature/preset and theme refinements. |

Exact per-version release dates are not restated here to avoid false precision;
see the repository's tags and CHANGELOG for the authoritative timeline[^4].

## References

[^1]: dandavison/delta README — features and bat theme/grammar sharing. https://github.com/dandavison/delta
[^2]: GitHub API, `repos/dandavison/delta` — 31,436 stars, 550 forks, ~407 open issues, MIT, created 2019-06-24, last push 2026-07-05 (fetched 2026-07-15).
[^3]: Delta installation guide. https://dandavison.github.io/delta/installation.html
[^4]: Delta releases and tags. https://github.com/dandavison/delta/releases

## Tags

rust, cli, git, diff, pager, syntax-highlighting, terminal, developer-tools, code-review, blame
