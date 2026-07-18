# zigtools/zls

> The community-maintained language server for Zig, written in Zig — the de facto
> IDE layer for a language whose compiler does not ship one.

[GitHub repo](https://github.com/zigtools/zls) ·
[Official website](https://zigtools.org/zls/install/) ·
[License: MIT](https://github.com/zigtools/zls/blob/master/LICENSE)

## Overview

ZLS is a non-official implementation of the Language Server Protocol for Zig,
started in April 2020 and maintained by the zigtools organization[^1]. Because
the Zig project itself ships no language server, ZLS is effectively the entire
Zig IDE story: VS Code, Neovim, Helix, Sublime, and Emacs setups all route
through it. Its ~5k stars understate its reach — nearly every Zig developer
with editor tooling runs it.

The defining tension is that ZLS reimplements Zig's semantics rather than
reusing the compiler's. Zig's meaning is dominated by `comptime` — types are
values computed at compile time — and ZLS performs its own approximate type
resolution instead of invoking the compiler's semantic analysis. The README is
candid that comptime and semantic analysis support is work-in-progress[^1].
Common patterns (generic containers like `ArrayList(T)`, payload captures,
custom packages) resolve well; deep comptime metaprogramming degrades to
unknown types. The long-discussed endgame is the Zig compiler providing
incremental semantic info to tooling directly; until then, ZLS is the interim
answer and has been for six years.

The second structural fact is version lockstep: Zig breaks syntax and std
freely between releases, so each tagged ZLS matches exactly one tagged Zig,
and ZLS master tracks Zig master (currently targeting a 0.17.0-dev
compiler)[^1]. Running mismatched versions is the #1 user-reported failure.

## Getting Started

Prebuilt binaries and per-editor setup live at the install guide[^2]; the
VS Code Zig extension can download a ZLS matching your Zig version
automatically. From source (requires a matching Zig):

```bash
git clone https://github.com/zigtools/zls
cd zls
zig build -Doptimize=ReleaseSafe
```

Neovim via nvim-lspconfig:

```lua
require("lspconfig").zls.setup({})
```

Full compile-error diagnostics are opt-in via build-on-save[^3], e.g. in
`zls.json`:

```json
{ "enable_build_on_save": true }
```

## Architecture / How It Works

ZLS is a single static binary written in Zig with no runtime dependencies.
Key components:

- **Parsing** — reuses the Zig standard library's self-hosted tokenizer and
  parser (`std.zig.Ast`), so syntax support is exact for the Zig version ZLS
  was built against. This is also why lockstep matters: new syntax in a newer
  Zig is a parse error to an older ZLS.
- **Semantic analysis** — ZLS's own analyser walks the AST and performs
  approximate type resolution, including limited comptime evaluation. It does
  not run the compiler's full `Sema`, trading correctness on comptime-heavy
  code for latency and independence from compiler internals[^1].
- **Build system introspection** — to resolve module imports and include
  paths, ZLS executes the project's `build.zig` with an injected build runner
  and reads back the dependency graph. Broken or side-effectful `build.zig`
  files therefore break navigation, not just builds.
- **Diagnostics** — default diagnostics are AST-level (parse errors and
  ast-check-style lints). Real compile errors require build-on-save, which
  shells out to `zig build` and parses its error output[^3].
- **Formatting** — delegates to the same rendering logic as `zig fmt`, so
  editor formatting and CI formatting cannot disagree.

LSP feature coverage is broad: completions, hover, go-to-definition, find
references, rename, workspace/document symbols, semantic tokens, inlay hints,
and code actions[^1].

## Production Notes

- **Keep Zig and ZLS in lockstep.** The README says it plainly: upgrade both
  together[^1]. Mismatches produce spurious errors, missing completions, or
  crashes. Teams pinning a Zig nightly must pin the corresponding ZLS nightly
  from the zigtools build index; tagged ZLS releases only work with the same
  tagged Zig.
- **Diagnostics are misleading out of the box.** Without build-on-save, a
  file with type errors can look clean because only AST-level checks run.
  Enable build-on-save for truth, but note it runs your full `zig build` on
  every save — on large projects budget for that, or scope the build step.
- **Comptime blind spots.** APIs built from functions returning types beyond
  common std patterns often lose completions and hover types. This is the
  known WIP area[^1], not a config problem; check upstream issues before
  filing.
- **C interop is partial.** Analysis through `@cImport` boundaries is limited
  compared to what the compiler itself resolves; mixed Zig/C codebases should
  expect weaker navigation on the C side.
- **Resource profile is light.** Compared to heavyweight servers like
  rust-analyzer, ZLS starts fast and uses modest memory — a deliberate
  consequence of not doing whole-program compiler-grade analysis.
- **Maintenance reality.** Actively developed (pushed within the last day as
  of July 2026; 143 open issues, 436 forks) by a small volunteer team funded
  via OpenCollective[^1]. It is community infrastructure, not a vendor
  product; breaking Zig releases can briefly leave tagged ZLS behind.

## When to Use / When Not

**Use when:**
- You write Zig in any LSP-capable editor — there is no comparable
  alternative, and the install cost is one binary.
- You want formatting, navigation, and rename that exactly match `zig fmt`
  and Zig's own parser.
- You track Zig master and can pair it with ZLS master builds.

**Avoid when:**
- You expect compiler-grade diagnostics by default — enable build-on-save or
  keep a `zig build` loop beside the editor.
- Your codebase is dominated by comptime metaprogramming and you would rely
  on the server for type truth; the compiler remains the only oracle.
- You cannot keep toolchain versions in sync (e.g. locked editor plugins in a
  managed environment) — a stale ZLS is worse than none on newer Zig.

## Alternatives

There is no mature drop-in replacement; these are adjacent options:

- ziglang/zig — the compiler's own `zig fmt`, `zig ast-check`, and `zig build`
  in a watch loop cover formatting and diagnostics without any server.
- ziglang/vscode-zig — the VS Code extension; not a replacement but the
  supported way to install and version-manage ZLS on that editor.
- llvm/llvm-project (clangd) — use alongside ZLS for the C/C++ side of mixed
  Zig/C codebases where ZLS's cross-language analysis stops.
- rust-lang/rust-analyzer — not for Zig, but the architectural benchmark; if
  compiler-grade IDE semantics are a hard requirement, that maturity today
  exists in Rust's toolchain, not Zig's.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-04 | Repository created; early releases versioned independently. |
| 0.10.0 | 2022-11 | Version lockstep with Zig releases established by this era. |
| 0.11.0 | 2023-08 | Tracks Zig 0.11; analyser rework period. |
| 0.12.0 | 2024-04 | Tracks Zig 0.12. |
| 0.13.0 | 2024-06 | Tracks Zig 0.13. |
| 0.14.0 | 2025-03 | Tracks Zig 0.14; build-on-save documented as the diagnostics path[^3]. |
| 0.15.x | 2025-08 | Tracks Zig 0.15. |
| 0.16.0 | 2026 | Current tagged line; master targets Zig 0.17.0-dev[^1]. |

## References

[^1]: ZLS README. https://github.com/zigtools/zls/blob/master/README.md
[^2]: Zigtools installation guide. https://zigtools.org/zls/install/
[^3]: Zigtools, "Build-On-Save" guide. https://zigtools.org/zls/guides/build-on-save/

## Tags

zig, language-server, lsp, developer-tools, ide-tooling, autocomplete, static-analysis, editor-integration, compiler-tooling
