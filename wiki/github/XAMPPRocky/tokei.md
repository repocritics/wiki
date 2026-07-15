# XAMPPRocky/tokei

> A fast command-line code counter that reports files, lines, code, comments, and blanks per language — a Rust successor to cloc.

[GitHub repo](https://github.com/XAMPPRocky/tokei) ·
[Crate](https://crates.io/crates/tokei) ·
[License: MIT OR Apache-2.0](https://github.com/XAMPPRocky/tokei/blob/master/LICENCE-MIT)

## Overview

Tokei ("時計", Japanese for clock) counts lines of code. Point it at a directory and it walks the tree, classifies each file by language, and prints a table of files, total lines, code, comments, and blanks — grouped by language[^1]. It is the most widely packaged of the modern cloc-style tools, available through nearly every OS package manager (Alpine, Arch, Fedora, Homebrew, Nix, winget, Scoop, and more), and it is also published as a Rust library on crates.io for embedding.

The defining tension is heuristic speed versus parser accuracy. Tokei does not build an AST; it scans each file with a per-language comment/string state machine derived from definitions in `languages.json`. That makes it fast enough to count millions of lines in seconds, and accurate enough to handle nested comments and comment-like sequences inside string literals[^2] — but it is still a line classifier, not a compiler front end. Ambiguous file extensions and generated files (the classic `.d` dependency-file false positive from gcc[^3]) can be miscategorised, and the "code vs comment vs blank" split is only as good as each language's definition.

The second thing to understand before relying on it: several output formats and the release cadence are not what a casual reader assumes. Serialization is feature-gated at compile time, and the project went roughly five years between tagged major releases before resuming in late 2025 (see Production Notes and History).

## Getting Started

```console
# pick your package manager
brew install tokei          # macOS
cargo install tokei         # any platform with Rust
pacman -S tokei             # Arch
winget install XAMPPRocky.tokei   # Windows
```

```console
$ tokei ./src
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Language      Files      Lines       Code   Comments     Blanks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Rust            19        3416       2840        116        460
 TOML             2          77         64          4          9
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

```console
# sort by code, count multiple trees, filter languages, emit JSON
$ tokei ./foo ./bar --sort code --type Rust,Markdown --output json
```

As a library (`Cargo.toml`: `tokei = "14"`):

```rust
use tokei::{Config, Languages};

let mut languages = Languages::new();
languages.get_statistics(&["src"], &[".git"], &Config::default());
let rust = &languages[&tokei::LanguageType::Rust];
println!("{} lines of Rust code", rust.code);
```

## Architecture / How It Works

Tokei's pipeline is three stages: **walk → classify → count**, parallelised across files.

- **Walk.** Directory traversal uses the `ignore` crate — the same walker ripgrep uses — so `.gitignore`, `.ignore`, `.tokeignore`, and VCS ignore files are honoured by default and traversal is multi-threaded. This is why `--no-ignore`, `--hidden`, and their finer-grained variants exist: the default view is deliberately the "tracked source" view, not "every byte on disk".
- **Classify.** Language detection is primarily by file extension (with some filename and shebang special-casing). The mapping and each language's comment/quote/nesting rules live in `languages.json`, which a build script compiles into Rust at build time rather than parsing at runtime[^2]. Adding a language means editing that JSON and regenerating.
- **Count.** Each file is scanned by a state machine that tracks line comments, block comments (including nested block comments), and string literals so that comment markers inside strings are not miscounted. Files are processed in parallel via a work-stealing thread pool. Tokei also detects embedded languages — e.g. Rust and Shell fenced inside Markdown, or JavaScript inside HTML — and reports them as child rows under a parent with a combined total.

Because classification is heuristic and definition-driven, tokei is fast and easy to extend but structurally unable to be exact for constructs its state machine doesn't model. It is a line accountant, not a semantic analyzer — it reports no complexity, no token counts, and no dependency information.

## Production Notes

**Serialization is feature-gated.** A plain `cargo install tokei` can produce a binary compiled *without* JSON/YAML/CBOR support; attempting `--output json` then prints a notice that the binary was built without serialization formats. To guarantee them, install with `cargo install tokei --features all` (or `--features json`/`yaml`/`cbor`). Distro-packaged builds vary in which features they enable, so if you script against tokei's JSON, pin how the binary was built rather than assuming[^1].

**Release cadence is bursty.** Tagged releases jumped from v12.1.0 (December 2020) to v13.0.0 (November 2025) — roughly five years between major tags — before v14.0.0 followed a month later[^4]. Development continued on `master` throughout, but teams pinning to published releases saw a long quiet period. If you depend on a recently added language definition, check whether it landed in a tagged release or only on `master`.

**`.gitignore` respect cuts both ways.** By default tokei will not count anything your VCS ignores. Generated code, vendored dependencies, and build output are usually excluded — often what you want, occasionally a silent undercount. Use `--no-ignore` for a raw count.

**Heuristic misclassification.** Extension collisions are the main footgun: gcc's `.d` dependency files register as the D language[^3]; some templating and config formats share extensions. The fix is `--exclude`/`.tokeignore` or `--type` filtering, not a config change. Treat the per-language split as a good estimate, not an audited figure — do not use it where an exact, defensible line count matters (billing, compliance).

**Dual license.** GitHub reports the license as `NOASSERTION` because the project is dual-licensed MIT OR Apache-2.0; downstream consumers may use either.

## When to Use / When Not

**Use when:**
- You want a fast, scriptable LOC breakdown across a large or polyglot repo.
- You need machine-readable output (JSON/YAML/CBOR) to feed dashboards, badges, or CI gates.
- You want to embed code counting inside a Rust program via the library API.
- You value broad packaging and cross-platform binaries over configurability.

**Avoid when:**
- You need exact, audit-grade counts or full tokenization — tokei is heuristic by design.
- You want complexity, COCOMO cost, or maintainability metrics — tokei only counts lines.
- Your language isn't in `languages.json` and you can't contribute a definition.
- You need semantic analysis, dependency graphs, or per-function metrics.

## Alternatives

- boyter/scc — the closest competitor; Go, comparable speed, and additionally estimates cyclomatic complexity and COCOMO cost. Use scc when you want complexity/cost estimates alongside counts.
- AlDanial/cloc — the original Perl tool; slower but long-established, with format conversions and diff modes tokei lacks. Use cloc when you need its specific report formats or its maturity.
- github/linguist — Ruby; language *detection and attribution* for GitHub's stats, not a counter. Use linguist when you care about repo language percentages, not line tables.
- cgag/loc — an older Rust cloc clone; simpler and less maintained. Use only if you want a minimal dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-05-26 | Repository created[^5]. |
| 10.0 | 2019-06-10 | Milestone release[^4]. |
| 11.0 | 2020-03-21 | Speed-focused release; README benchmarks reference it[^1]. |
| 12.0 | 2020-06-22 | Major release[^4]. |
| 12.1 | 2020-12-23 | Last tag before a multi-year release gap[^4]. |
| 13.0 | 2025-11-25 | First major tag in ~5 years; development resumes[^4]. |
| 14.0 | 2025-12-30 | Current release[^4]. |

## References

[^1]: tokei README — features, usage, installation, and output formats. https://github.com/XAMPPRocky/tokei/blob/master/README.md
[^2]: Language definitions and build-time code generation — `languages.json` and the crate's counting model. https://github.com/XAMPPRocky/tokei/blob/master/languages.json
[^3]: tokei README, "Common issues" — gcc `.d` files misdetected as the D language. https://github.com/XAMPPRocky/tokei/blob/master/README.md#common-issues
[^4]: tokei releases — tags and publish dates (v10.0.0 2019-06-10 through v14.0.0 2025-12-30). https://github.com/XAMPPRocky/tokei/releases
[^5]: GitHub repository metadata — created 2015-05-26. https://github.com/XAMPPRocky/tokei

## Tags

rust, cli, code-counting, sloc, cloc, static-analysis, developer-tools, command-line-tool, metrics, cross-platform
