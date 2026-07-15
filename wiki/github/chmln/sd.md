# chmln/sd

> Find & replace CLI with JavaScript/Python-style regex — a narrow, opinionated alternative to sed's substitution command.

[GitHub repo](https://github.com/chmln/sd) ·
[License: MIT](https://github.com/chmln/sd/blob/master/LICENSE)

## Overview

`sd` ("search & displace") is a single-purpose command-line find-and-replace tool written in Rust, first published in late 2018[^1]. It deliberately does one thing — substitute text — where `sed` is a full stream-editing language with addressing, hold space, and a scripting model. The pitch is ergonomics: you write the pattern and the replacement as two separate arguments (`sd before after`) instead of packing them into one delimiter-escaped expression (`sed 's/before/after/g'`), and the regex dialect is the one you already know from JavaScript and Python rather than POSIX BRE/ERE.

Two design choices define it. First, replacement is global by default — there is no `/g` flag, because substituting every match is the common case. Second, the regex engine is Rust's `regex` crate, which guarantees linear-time matching but, as a direct consequence, has **no backreferences and no lookahead/lookbehind**[^2]. This is the central tradeoff: `sd` is safe against catastrophic backtracking and fast on large inputs, but it cannot express a class of patterns that PCRE, Perl, and GNU `sed` handle. For those, `sd` is the wrong tool.

`sd` is not a `sed` replacement in scope — it does not do line addressing, deletion, insertion, or transliteration. It is a replacement for the specific, dominant `s///` use case, and it targets developers doing interactive edits and codebase-wide substitutions rather than programmatic stream pipelines.

## Getting Started

```bash
cargo install sd
# or via a package manager: brew install sd, apt install sd,
# pacman -S sd, etc. (see repology for distro coverage)
```

```sh
# Regex is the default. Replacement is global — no /g needed.
echo 'lorem ipsum 23   ' | sd '\s+$' ''        # trim trailing whitespace

# Capture groups use $1 / ${1} (JS/Python style), not \1
echo '123.45' | sd '(?P<d>\d+)\.(?P<c>\d+)' '$d dollars and $c cents'

# String-literal mode: no escaping of regex metacharacters
echo 'a.b.c' | sd -F '.' '/'                    # a/b/c

# In-place file edit (modifies the file directly)
sd 'window.fetch' 'fetch' http.js

# Preview without writing
sd -p 'window.fetch' 'fetch' http.js
```

Recursion is not built in. `sd` operates on the files you name (or stdin); to sweep a tree you compose it with a file finder:

```sh
fd --type file --exec sd 'from "react"' 'from "preact"'
```

## Architecture / How It Works

`sd` is a thin, intentionally small binary. Its behavior reduces to: parse args, build a `regex::Regex` (or a literal matcher under `-F`), read input, call the regex crate's `replace_all`, write output. The value is in the defaults and the argument shape, not in a novel engine.

**Regex engine.** Matching is delegated wholesale to the `regex` crate — the same engine behind ripgrep. It is a finite-automaton implementation with no backtracking, which is why lookaround and backreferences are absent by construction rather than by omission. Replacement string syntax (`$1`, `${name}`, `$$` to emit a literal `$`) is the regex crate's own, which is why it looks like JavaScript and not like `sed`'s `\1`.

**Processing modes.** By default `sd` processes input **line by line**: only one line is held in memory, output can stream before EOF, and `^`/`$` anchor to line boundaries so `\s+$` trims trailing whitespace without consuming newlines. Patterns that must span line boundaries (matching or emitting `\n`, multi-line constructs) require the `-A` / `--across` flag, which reads the whole input into memory and matches against it as one string. The two modes are a genuine memory-versus-capability split: across mode buffers the entire input, line mode does not.

**In-place editing.** When given filenames, `sd` rewrites each file with its substituted contents. There is no `-i`-style backup-suffix option; the README's recommended safety net is version control or an explicit copy before editing (`cp {} {}.bk` in an `fd --exec` chain). Argument parsing treats any token starting with `-` as a flag, so replacements that begin with a dash need the `--` end-of-options separator (`sd 'foo' -- '-w'`).

## Production Notes

**The regex-crate limitation is the thing that will surprise you.** Any workflow that relies on backreferences (`\1` inside the *pattern*) or lookahead/lookbehind will not port from `sed`/`perl`/`grep -P` to `sd`. There is no flag to enable them — the engine does not implement them. Confirm your patterns fit the linear-time subset before scripting `sd` into anything.

**Line-by-line vs across is a correctness switch, not just performance.** In default line mode, a pattern containing `\n` matches nothing, silently, because the newline never appears within a single processed line. The fix is `-A`, but `-A` also changes memory behavior: the project's own benchmarks show across mode holding ~74 MB peak RSS on a ~36 MB input versus ~3 MB for line mode[^3]. For multi-gigabyte files, prefer line mode and patterns that stay within a line.

**Performance is real but scenario-dependent.** On large inputs the README reports `sd` running roughly 2–12× faster than GNU `sed` for simple and regex substitutions[^3]; the wide range reflects that `sed`'s BRE engine backtracks where the regex crate does not. Note the inversion: `sd` in *line-by-line* mode can be slightly slower than `sed` for a trivial literal replace, because per-line overhead dominates when there is no regex work to amortize. Treat the "N× faster" numbers as workload-specific, not a blanket claim.

**In-place edits are not transactional across a set.** `sd a b *.txt` edits each file in turn; an interruption mid-run leaves some files rewritten and others not. There is no dry-run-then-commit for a batch — `-p` previews to stdout for a single invocation only. Run destructive sweeps under version control so the diff is reviewable and revertible.

**No recursion, globbing, or file filtering.** This is deliberate (Unix composition), but it means every tree-wide edit is really an `fd`/`find` command with `sd` on the end, and you own the file-selection correctness. A too-broad `fd` pattern piped into an in-place `sd` will happily rewrite binaries and generated files.

## When to Use / When Not

**Use when:**
- You want fast, global find-and-replace with a familiar regex dialect and no delimiter escaping.
- You're doing interactive one-off substitutions or codebase sweeps (paired with `fd`).
- Your patterns fit a linear-time regex (no backreferences, no lookaround).
- You value the safety of an engine that cannot catastrophically backtrack.

**Avoid when:**
- You need backreferences or lookahead/lookbehind — `sd` cannot express them.
- You need the rest of `sed`'s language: address ranges, deletion, insertion, hold space, transliteration.
- You want syntax-aware (AST-level) rewrites — regex is the wrong abstraction for structural refactors.
- You need atomic/backup-guaranteed in-place editing without relying on external version control.

## Alternatives

- GNU sed — use when you need the full stream-editing language (addressing, `d`/`i`/`a`, hold space) or backreference-dependent patterns `sd` cannot express.
- BurntSushi/ripgrep — use `rg --passthru`/`--replace` when you want tree-wide, preview-to-stdout replacement without in-place writes; shares the same regex engine as `sd`.
- facebookincubator/fastmod — Rust codemod tool built in the same spirit as `sd`; use when you want interactive per-hunk confirmation across a large refactor.
- ast-grep/ast-grep — use when the edit is structural and should respect syntax (rename a call, not a string), rather than regex-matched.
- comby-tools/comby — use for language-aware structural search-and-replace with balanced-delimiter matching that regex handles poorly.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-12 | First public release; Rust `regex`-backed find & replace[^1]. |
| 0.x | 2019–2023 | Long 0.x series; `-F` literal mode, `-p` preview, in-place editing, package-manager distribution. |
| 1.0.0 | 2023 | First stable major release. |
| — | 2026-02 | Most recent pushed commit at time of writing[^4]. |

## References

[^1]: chmln/sd repository, created 2018-12-23. https://github.com/chmln/sd
[^2]: Rust `regex` crate documentation — states the engine has no backreferences or lookaround as a condition of its linear-time guarantee. https://docs.rs/regex/latest/regex/#syntax
[^3]: chmln/sd README, "Benchmarks" section (hyperfine results and peak-RSS table). https://github.com/chmln/sd#benchmarks
[^4]: GitHub API `repos/chmln/sd`, `pushed_at` 2026-02-25 (fetched 2026-07-15). https://api.github.com/repos/chmln/sd

## Tags

rust, cli, find-and-replace, regex, text-processing, command-line, sed-alternative, terminal, developer-tools
