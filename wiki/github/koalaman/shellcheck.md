# koalaman/shellcheck

> The static analysis tool for shell scripts — a bash/sh/dash/ksh parser in Haskell that catches the quoting bugs and portability landmines shells silently accept.

[GitHub repo](https://github.com/koalaman/shellcheck) ·
[Official website](https://www.shellcheck.net) ·
[License: GPL-3.0](https://github.com/koalaman/shellcheck/blob/master/LICENSE)

## Overview

ShellCheck is a linter for POSIX sh, bash, dash, and ksh scripts, started by Vidar Holen in November 2012[^1]. It reimplements a shell grammar from scratch in Haskell rather than reusing any shell's own parser, which lets it parse *almost*-valid scripts and explain precisely why they will misbehave — the class of bug shells accept silently (unquoted `$var` word splitting, `[ $a == b ]` bashisms under `#!/bin/sh`, `for f in $(ls)`). Each finding carries a stable code (SC1xxx syntax, SC2xxx semantics, SC3xxx portability) with a dedicated wiki page, which makes diagnostics unusually actionable and drove wholesale CI adoption — it ships preinstalled on GitHub Actions Linux runners, Travis CI, Codacy, Code Climate, and CircleCI (via orb)[^1].

The defining tension: ShellCheck is a *static* analyzer for a language that is dynamic to its core. It cannot see through `eval`, dynamically constructed commands, or variable indirection, and it never executes anything. It trades runtime coverage for zero-risk analysis and gets remarkably far — the famous Steam `rm -rf "$STEAMROOT/"*` home-directory deletion bug is exactly the pattern its robustness checks flag[^2]. A second tension is sociological: the tool for shell scripters is written in Haskell, which keeps the contributor pool small. It is effectively a single-maintainer project with roughly yearly releases, though alive (last push 2026-06; ~39.7k stars, ~1.9k forks); the ~1,100 open issues are mostly check proposals and false-positive reports, not abandonment.

## Getting Started

```bash
# macOS
brew install shellcheck
# Debian/Ubuntu
sudo apt install shellcheck
# or a statically linked binary from GitHub Releases (Linux/macOS/Windows)
```

```sh
#!/bin/bash
# deploy.sh — three real bugs ShellCheck catches
for f in $(ls *.txt); do        # SC2045: iterating over ls output
  cp $f /backup                 # SC2086: unquoted $f word-splits
done
[ $# > 2 ] && echo "too many"   # SC2071: > is redirection inside [ ]
```

```bash
shellcheck deploy.sh              # human-readable, colored
shellcheck -f gcc deploy.sh       # editor/CI-friendly one-line format
shellcheck -f diff deploy.sh | git apply   # apply auto-fixes where available
```

In CI, the exit code is nonzero when findings meet the threshold; use `--severity=warning` to ignore style-level notes.

## Architecture / How It Works

ShellCheck is a pipeline of a hand-written parser and a large catalog of AST checks, all in Haskell:

1. **Parsing** — a parser-combinator (Parsec-based) grammar covering a superset of sh/bash/dash/ksh. Because it is not any shell's real parser, it can keep going on broken input and emit SC1xxx diagnostics that explain the syntax error instead of bash's notorious `unexpected token` messages.
2. **Dialect resolution** — the target shell comes from the shebang, the `--shell` flag, or a `# shellcheck shell=sh` directive. The same script produces different findings per dialect: SC3xxx bashism warnings only fire when the declared shell is POSIX sh[^3].
3. **Checks** — hundreds of pattern-matching passes over the AST, each keyed to an SC code. Optional opinionated checks (e.g. `require-variable-braces`, `quote-safe-variables`) are off by default and enabled via `--enable` or `.shellcheckrc`.
4. **Dataflow analysis** — since 0.9.0 a control-flow-graph engine powers deeper checks such as SC2317 (unreachable code)[^4]. This is the expensive pass; 0.10.0 added `--extended-analysis=false` as an escape hatch[^5].
5. **Formatters** — tty (colored), `gcc`, `checkstyle` XML, `json`/`json1`, `diff` (unified-diff auto-fixes), `quiet`. The stable formats are the integration surface for the editor plugins (vscode-shellcheck, ALE, Flycheck) and CI wrappers.

Inline control is done with comment directives: `# shellcheck disable=SC2086`, `shell=`, `source=`, `source-path=`, `enable=`. Sourced files are *not* followed by default (SC1091 notes this); `-x` plus `source-path` directives opt in. The core also ships as a Haskell library on Hackage, which is how shellcheck.net (always synced to latest git) embeds it[^1].

## Production Notes

**Pin your version in CI.** The README itself warns that new releases add new checks, which means a routine version bump can fail previously-green pipelines[^1]. Distro packages lag significantly (some LTS distros shipped 0.4.x for years); for reproducible CI, download a pinned static binary or use the `koalaman/shellcheck:v0.x.y` Docker tag, not `:stable`.

**Blind spots are silent.** `eval`, command names held in variables, `declare -n` indirection, and anything constructed at runtime pass through unanalyzed — no warning that analysis was skipped. Teams that lint-gate merges sometimes conclude a script is "clean" when it is merely opaque.

**Sourcing needs configuration.** Scripts that `source lib/common.sh` produce SC1091 and lose all knowledge of sourced variables/functions unless you run with `-x` and correct `source-path` directives. In monorepos where CI's working directory differs from developers', this is the most common source of CI-vs-local disagreement.

**Performance is fine until it isn't.** Typical scripts lint in milliseconds, but the post-0.9.0 dataflow pass can crawl on multi-thousand-line scripts or enormous `case` statements; `--extended-analysis=false` restores speed at the cost of the CFG-based checks[^5].

**Scope limits.** bash/sh/dash/ksh only — zsh and fish are out of scope, and zsh scripts mislabeled `#!/bin/bash` produce nonsense findings. Auto-fix (`-f diff`) covers only a subset of checks; it is not a formatter — pair with mvdan/sh's shfmt. Prefer scoped inline `disable=` comments with justification over blanket codes in `.shellcheckrc`, or the suppression list quietly becomes a list of real bugs you no longer see.

**Building from source** requires GHC/Cabal and about 2 GB of RAM[^1] — in practice nobody does; the statically linked release binaries are the distribution mechanism.

## When to Use / When Not

**Use when:**
- Any repository contains shell scripts touched by more than one person — the cost is near zero and SC2086-class bugs are ubiquitous.
- You gate CI on script quality: exit codes, `gcc`/`checkstyle`/JSON output, and preinstalled availability on major CI platforms make integration trivial.
- You maintain `#!/bin/sh` scripts that must run on dash/BusyBox — the SC3xxx portability checks are the best automated bashism detector available.
- You are teaching shell: diagnostics link to per-code wiki explanations.

**Avoid when:**
- Your scripts are zsh or fish — unsupported, and results will mislead.
- Scripts are heavily metaprogrammed (`eval`, generated commands) — coverage collapses silently and the green check gives false confidence.
- You want formatting or style normalization — that is shfmt's job.
- Your scripts are long enough to be programs: past a few hundred lines the correct fix is usually Python/Go, not a better-linted bash script.

## Alternatives

- mvdan/sh — shfmt formatter plus a Go shell parser; use it alongside (for formatting) or instead when you need a shell parser as a library in Go.
- anordal/shellharden — corrective rewriter that mechanically hardens quoting; use when you want automated fixes for quoting specifically, not a linter.
- checkbashisms (Debian devscripts, not a GitHub project) — narrow bashism detector for `#!/bin/sh`; use in Debian packaging workflows where it is already standard.
- bats-core/bats-core — runtime testing for shell scripts; complements rather than replaces static analysis when behavior matters more than form.
- oils-for-unix/oils — a different bet entirely: replace bash with a stricter shell (OSH/YSH) instead of linting the old one.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-11 | Repository created by Vidar Holen (koalaman)[^6]. |
| 0.7.0 | 2019-07 | `.shellcheckrc` support, optional checks, `-f diff` auto-fix output[^3]. |
| 0.7.2 | 2021-04 | Generic bashism warning SC2039 split into granular SC3xxx portability codes[^3]. |
| 0.8.0 | 2021-11 | Incremental release; new checks and directive improvements[^3]. |
| 0.9.0 | 2022-12 | Control-flow-graph dataflow engine; SC2317 unreachable-code detection[^4]. |
| 0.10.0 | 2024-03 | `--extended-analysis` flag to opt out of the slower dataflow pass[^5]. |
| 0.11.0 | 2025-08 | Current stable release[^3]. |

## References

[^1]: ShellCheck README. https://github.com/koalaman/shellcheck#readme
[^2]: steam-for-linux issue #3671, "Moved ~/.local/share/steam. Ran steam. It deleted everything" — 2015-01. https://github.com/ValveSoftware/steam-for-linux/issues/3671
[^3]: ShellCheck CHANGELOG. https://github.com/koalaman/shellcheck/blob/master/CHANGELOG.md
[^4]: ShellCheck wiki, SC2317. https://www.shellcheck.net/wiki/SC2317
[^5]: ShellCheck v0.10.0 release notes. https://github.com/koalaman/shellcheck/releases/tag/v0.10.0
[^6]: GitHub API, repository metadata (created_at 2012-11-17). https://api.github.com/repos/koalaman/shellcheck

## Tags

haskell, shell, bash, linter, static-analysis, developer-tools, cli, posix, ci, code-quality
