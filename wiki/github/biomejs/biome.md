# biomejs/biome

> A Rust toolchain for web projects — one binary that formats and lints JavaScript, TypeScript, JSX, JSON, CSS, and GraphQL, positioned as a fast replacement for Prettier + ESLint.

[GitHub repo](https://github.com/biomejs/biome) ·
[Official website](https://biomejs.dev) ·
[License: MIT OR Apache-2.0](https://github.com/biomejs/biome/blob/main/LICENSE-MIT)

## Overview

Biome is a single-binary formatter and linter for front-end code, written in Rust. It emerged in August 2023 as a community fork of Rome Tools (`rome`), the all-in-one toolchain started by Babel author Sebastian McKenzie; when Rome Tools Inc. ran out of funding and laid off its engineers, core contributors forked the codebase under a new name and governance model[^1]. Biome deliberately narrowed the original Rome ambition — which had aimed to also be a bundler, compiler, and test runner — down to the two jobs teams most wanted to consolidate: formatting and linting.

The pitch is consolidation and speed. Instead of running Prettier for formatting and ESLint (plus typescript-eslint and a stack of plugins) for linting, Biome does both from one dependency with one config file and no Node.js requirement at runtime. It advertises 97% output compatibility with Prettier[^2] and ships more than 500 lint rules ported from ESLint, typescript-eslint, and other sources[^3]. As of 2026 it has roughly 25.4k stars and is actively maintained, with a community-elected core team funded through Open Collective and GitHub Sponsors.

The defining tradeoff is scope versus reach. Biome is fast and cohesive precisely because it does not embed the TypeScript compiler and does not support the long tail of Prettier plugins (Vue SFCs, Svelte, Astro, Markdown, YAML, PHP). That makes it excellent for the JS/TS/JSON/CSS core and awkward the moment your codebase needs a format or a type-aware lint rule that lives outside that core.

## Getting Started

```shell
npm install --save-dev --save-exact @biomejs/biome
npx @biomejs/biome init   # writes biome.json
```

```shell
# format files in place
npx @biomejs/biome format --write

# lint and apply safe fixes
npx @biomejs/biome lint --write

# format + lint + organize imports, apply safe fixes
npx @biomejs/biome check --write

# check-only, non-zero exit on issues (for CI)
npx @biomejs/biome ci
```

A minimal `biome.json`:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "formatter": { "enabled": true, "indentStyle": "space" },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  }
}
```

Biome has sane defaults and runs with zero configuration. Migration helpers convert existing setups: `biome migrate eslint` and `biome migrate prettier` translate rule config and formatter options into `biome.json`.

## Architecture / How It Works

Biome is built on a lossless concrete syntax tree (CST) modeled on rust-analyzer's red/green tree design — its `biome_rowan` crate is a fork of that library[^4]. The CST preserves every byte of the source (whitespace, comments, even malformed input) and has strong error recovery, which is what lets Biome format and lint code that does not fully parse, and what makes it suitable for editor-interactive use.

The pipeline has three main stages:

1. **Parse** — source text → full-fidelity CST per language. Each language (JS/TS/JSX, JSON, CSS, GraphQL) has its own parser producing a shared tree shape.
2. **Format** — the CST is lowered into a formatter intermediate representation (a document IR of groups, lines, and indents, conceptually similar to Prettier's Doc), then printed back to a target width. Prettier's algorithm is reimplemented, not wrapped, which is the source of the ~3% divergence.
3. **Analyze** — lint rules and assist actions run as queries over the CST plus a semantic model (scope/binding resolution). Rules are declarative visitors; each can emit diagnostics and optional code actions classified as safe or unsafe fixes.

Editor integration runs through a **daemon**: `biome lsp-proxy` talks to a long-lived workspace server so repeated requests reuse parsed state and caching. The LSP is first-party, not a wrapper around the CLI. Everything is parallelized across files (via a work-stealing thread pool), which is where most of the speed advantage over the Node-based toolchain comes from.

Historically Biome's analyzer was intentionally type-unaware: unlike typescript-eslint it did not invoke `tsc`, trading away type-dependent rules for speed. Biome 2.0 (2025) added its own lightweight type inference engine — independent of the TypeScript compiler — enabling a first set of type-aware rules, along with GritQL-based plugins and monorepo/nested-config support[^5]. The inference is deliberately partial and does not match the fidelity of a full type checker.

## Production Notes

**Type-aware linting is limited.** If your team relies on typescript-eslint rules that require full type information (e.g. `no-floating-promises`, `no-unsafe-*`), Biome's own inference covers only a subset. Do not assume a one-to-one migration off typescript-eslint; audit which rules you actually depend on first.

**Language coverage is the core, not the long tail.** Biome formats and lints JS/TS/JSX/TSX, JSON/JSONC, CSS, and GraphQL well. HTML and Markdown support is partial/experimental, and framework single-file components (Vue, Svelte, Astro) are not covered the way Prettier plugins cover them. Teams with `.vue`/`.svelte`/`.astro` files typically keep Prettier for those and use Biome for everything else — which undercuts the "one tool" benefit.

**Rule parity with ESLint is incomplete.** 500+ rules is substantial, but specific ESLint plugins (certain import-resolution, framework-specific, or bespoke org rules) may have no equivalent. There is a plugin path via GritQL since 2.0, but it is far narrower than authoring a JS ESLint rule. Check that your must-have rules exist before committing.

**Config semantics changed at 2.0.** The 1.x `include`/`ignore` keys were reworked into `includes` with different glob semantics, and import-organizing moved from the linter into a separate "assist" domain. `biome migrate` handles most of this, but hand-written configs and CI invocations often need manual review during the 1.x → 2.0 jump. Pin the exact version (`--save-exact`) — formatter output can shift subtly between minors, producing noisy diffs across a team on mixed versions.

**`check` vs `ci` vs individual commands.** `biome ci` is the non-mutating, fail-on-diff entry point for pipelines; `check --write` is the local fix-everything command. Running `format --write` alone will not apply lint fixes or organize imports. Mixing these up is the most common first-week confusion.

**Distribution.** The `@biomejs/biome` npm package pulls a platform-specific prebuilt binary (`@biomejs/cli-*`) as an optional dependency; there is no Node runtime cost, but locked-down CI that blocks optional dependencies or unusual CPU/OS targets can fail to resolve the binary. Standalone binaries and a WASM playground exist as alternatives.

## When to Use / When Not

**Use when:**
- Your codebase is primarily JS/TS/JSX/JSON/CSS and you want to collapse Prettier + ESLint into one fast dependency.
- You want zero-config defaults and a first-class LSP with format/lint on malformed code.
- CI lint/format time is a real bottleneck and you want Rust-level throughput.
- You want a single `biome.json` instead of maintaining `.eslintrc` + `.prettierrc` + plugin versions.

**Avoid when:**
- You depend on type-aware typescript-eslint rules or a large set of ESLint plugins with no Biome equivalent.
- You need to format Vue/Svelte/Astro SFCs, Markdown, YAML, or other Prettier-plugin languages as first-class citizens.
- You have heavy investment in custom ESLint rules written in JavaScript.
- Byte-identical Prettier output is a hard requirement (Biome targets ~97%, not 100%).

## Alternatives

- prettier/prettier — use instead when you need the widest language and plugin coverage (Vue, Svelte, Astro, Markdown, YAML, PHP) or exact-Prettier output.
- eslint/eslint — use instead when you need type-aware rules, custom JS-authored rules, or the full third-party plugin ecosystem.
- oxc-project/oxc — the `oxlint` Rust linter is even faster and stricter on speed; use when you want maximum lint throughput and format elsewhere.
- dprint/dprint — use instead when you want a pluggable multi-language formatter (Wasm plugins) rather than a fixed formatter + linter bundle.
- rome/tools — the archived predecessor; Biome is its live successor, so use Biome instead in all cases.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2023-08 | Community fork of Rome Tools after its funding collapse[^1]. |
| 1.0 | 2023-08 | First stable Biome release: formatter + linter, single binary. |
| 1.x | 2023–2025 | Rule additions, CSS/GraphQL support, `migrate` commands, broader editor extensions. |
| 2.0 | 2025-06 | Type inference engine (no `tsc`), GritQL plugins, monorepo/nested configs, assist domain, `includes` config rework[^5]. |

## References

[^1]: Biome announcement / naming, "Biome, a community successor to Rome" — biomejs.dev blog, 2023. https://biomejs.dev/blog/annoucing-biome/
[^2]: Prettier-compatibility challenge results (Algora), referenced by Biome README. https://algora.io/challenges/prettier
[^3]: Biome linter rule reference (500+ rules). https://biomejs.dev/linter/rules/
[^4]: Biome internals / architecture (CST built on a rust-analyzer-derived rowan library). https://biomejs.dev/internals/architecture/
[^5]: Biome 2.0 release notes ("Biotype") — type-aware analysis, plugins, monorepo support. https://biomejs.dev/blog/biome-v2/

## Tags

rust, javascript, typescript, formatter, linter, web-tooling, static-analysis, lsp, cli, prettier-alternative, eslint-alternative
