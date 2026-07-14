# rust-lang/rust-analyzer

> A from-scratch Rust compiler front-end built for editors — the language server behind almost every modern Rust IDE experience.

[GitHub repo](https://github.com/rust-lang/rust-analyzer) ·
[Official website](https://rust-analyzer.github.io/) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/rust-analyzer/blob/master/LICENSE-APACHE)

## Overview

rust-analyzer is a Language Server Protocol (LSP) implementation that provides IDE features for Rust: completion, go-to-definition, find-all-references, hovers, inlay hints, refactorings, and diagnostics[^1]. It works with any LSP client — VS Code (via the official `rust-analyzer` extension), Neovim, Emacs, Zed, Helix, and others. It has been the recommended and de facto standard Rust IDE backend since it superseded the older RLS, which was formally deprecated in 2022[^2].

The defining architectural decision is that rust-analyzer is a **separate compiler front-end from `rustc`**, not a wrapper around it. It re-implements lexing, parsing, macro expansion, name resolution, and type inference independently, tuned for the editor's constraints: incrementality (re-analyze only what changed on each keystroke), resilience (produce useful results from syntactically invalid, half-typed code), and latency (answer within tens of milliseconds). This is the project's central tension. Editors need a tolerant, always-on, incremental analyzer; a batch compiler like `rustc` is architecturally the opposite. The cost is two implementations of Rust's semantics that can and do diverge — rust-analyzer occasionally accepts code `rustc` rejects, or vice versa. The long-term hope is "library-ification" of `rustc` so both share components, but as of 2026 they remain largely separate front-ends.

rust-analyzer is a rust-lang project (moved into the org around 2020) and ships weekly releases every Monday rather than semver-tagged versions[^3].

## Getting Started

The usual path is the editor extension, which downloads the server binary automatically:

```
# VS Code: install the "rust-analyzer" extension (publisher: rust-lang).
# It fetches the matching server binary on first launch.
```

Manual install of the standalone server (for other editors):

```bash
rustup component add rust-analyzer
# binary path:
rustup which --toolchain stable rust-analyzer
```

Minimal client config (Neovim, via `nvim-lspconfig`):

```lua
require("lspconfig").rust_analyzer.setup({
  settings = {
    ["rust-analyzer"] = {
      cargo = { allFeatures = true },
      check = { command = "clippy" },   -- use clippy for on-save diagnostics
    },
  },
})
```

rust-analyzer analyzes real Cargo projects; it discovers structure via `cargo metadata` and expects a `Cargo.toml`. Non-Cargo builds require a hand-written `rust-project.json`.

## Architecture / How It Works

The system is built as a stack of libraries rather than a monolith[^4]:

1. **Syntax layer (`rowan`)** — a lossless, red-green syntax tree modeled on the approach used by Roslyn[^5]. Trees preserve every token, including whitespace and comments, and every node knows its exact source range. The parser is error-resilient: invalid input still yields a tree with error nodes, which is what makes completion-while-typing possible.
2. **Incremental computation (`salsa`)** — all analysis is expressed as memoized, demand-driven queries. When a file changes, salsa invalidates only the derived queries that transitively depended on it and recomputes lazily on demand[^6]. This is the engine behind sub-keystroke responsiveness.
3. **Semantic layer (`hir`)** — name resolution, macro expansion (declarative and procedural), and type inference. Trait solving historically runs through **chalk**, a separate logic-programming engine for Rust's trait system[^7].
4. **IDE layer** — features (completion, assists, diagnostics) implemented on top of the semantic queries, then adapted to LSP messages in the `rust-analyzer` binary.

**Procedural macros** are the hardest part. rust-analyzer cannot interpret proc-macros; it must compile them to dynamic libraries and execute them in a separate **proc-macro server** process. The ABI between the server and the loaded proc-macro dylib must match the toolchain that built them, which is a recurring source of "proc macro not expanded" failures after toolchain changes.

Because rust-analyzer implements its own type checker, it is not a thin `cargo check` wrapper. On-save diagnostics from `rustc`/`clippy` are separate: rust-analyzer runs an external `cargo check`/`cargo clippy` and surfaces its output, while flow-through IDE diagnostics (unresolved names, type mismatches shown live) come from its own analysis. Users see two diagnostic sources that can disagree.

## Production Notes

**Memory and first-load cost.** On large workspaces rust-analyzer can consume multiple GB of RAM, and the initial indexing (running `cargo metadata`, building proc-macros, populating salsa) can take from seconds to minutes before completions are accurate. On very large monorepos this is the dominant complaint.

**`cargo check` on save is the default diagnostic path** and it can be expensive — a save triggers a full workspace check, which competes with your own `cargo build`/`cargo run` for the target directory lock, causing stalls. Common mitigations: point check at a separate target dir, switch `check.command` to `clippy` only when you want it, or disable check-on-save and rely on live diagnostics.

**Proc-macro and build-script footguns.** Heavy proc-macro crates (serde derive, async runtimes, `sqlx::query!`) require the proc-macro server and build-script execution to be enabled; when the toolchain used to build them mismatches the server, expansion silently fails and downstream code shows spurious errors. Reloading the workspace or aligning toolchains is the usual fix.

**Divergence from `rustc`.** Because it is a second implementation, edge cases (const generics, complex trait bounds, new language features) may lag or differ. Treat rust-analyzer's live errors as a strong hint, not ground truth — a clean `cargo build` is authoritative.

**Configuration surface is large.** There are hundreds of settings (`rust-analyzer.cargo.*`, `.check.*`, `.procMacro.*`, `.files.excludeDirs`). Non-standard build systems (Bazel, Buck) need a generated `rust-project.json`; without it, discovery fails.

**Weekly releases** mean the VS Code extension updates often; the pre-release channel tracks nightly server builds and can introduce regressions. Pinning the server version is possible but manual.

## When to Use / When Not

**Use when:**
- You write Rust in any editor and want completion, refactors, and inline type info — this is the standard, effectively unavoidable choice.
- You need live, keystroke-latency feedback on partial code that a batch compiler cannot give.
- You want a single backend that works across VS Code, Neovim, Emacs, Zed, and Helix.

**Avoid / look elsewhere when:**
- You only need CI-grade correctness checking — use `cargo check` / `cargo clippy` directly; the language server adds no value in a headless pipeline.
- Your build is non-Cargo and you cannot produce a `rust-project.json` — discovery will not work.
- You are on a severely memory-constrained machine analyzing a huge workspace — indexing cost may be prohibitive.

## Alternatives

- rust-lang/rls — the original Rust Language Server; deprecated and archived in favor of rust-analyzer. Historical only.
- intellij-rust/intellij-rust — JetBrains' independent Rust front-end; use when you live in IntelliJ/CLion and want that platform's tooling instead of LSP.
- Use `cargo check` / rust-lang/rust-clippy directly when you want authoritative diagnostics in CI rather than an interactive editor backend.
- clangd / other-language LSPs — only relevant as the closest architectural analogue; not a Rust option.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2017-12 | Started as the `libsyntax2` parsing experiment by Aleksey Kladov (matklad)[^4]. |
| Renamed rust-analyzer | 2019 | Grows from parser experiment into a full LSP server with VS Code extension. |
| Moved into rust-lang org | ~2020 | Becomes the officially blessed successor to RLS[^2]. |
| RLS deprecated | 2022 | rust-analyzer designated the standard Rust language server[^2]. |
| `rustup component` distribution | 2022+ | Server shippable via `rustup component add rust-analyzer`. |
| Ongoing | weekly | Released every Monday; changelog at "This Week in rust-analyzer"[^3]. |

## References

[^1]: rust-analyzer README and manual. https://rust-analyzer.github.io/book/
[^2]: rust-lang/rls — deprecation in favor of rust-analyzer. https://github.com/rust-lang/rls
[^3]: "This Week in rust-analyzer" changelog (weekly releases). https://rust-analyzer.github.io/thisweek
[^4]: rust-analyzer manual, "Architecture". https://rust-analyzer.github.io/book/contributing/architecture.html
[^5]: rowan — lossless syntax trees library. https://github.com/rust-analyzer/rowan
[^6]: salsa — incremental computation framework. https://github.com/salsa-rs/salsa
[^7]: chalk — trait-solving engine for Rust. https://github.com/rust-lang/chalk

## Tags

rust, language-server, lsp, ide, developer-tooling, compiler-frontend, code-completion, static-analysis, salsa, incremental-computation
