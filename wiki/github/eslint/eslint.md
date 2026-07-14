# eslint/eslint

> The pluggable linter for JavaScript and JSX — every rule is a plugin, and the AST is the API.

[GitHub repo](https://github.com/eslint/eslint) ·
[Official website](https://eslint.org) ·
[License: MIT](https://github.com/eslint/eslint/blob/main/LICENSE)

## Overview

ESLint is a static analysis tool that finds and (where possible) fixes problematic patterns in ECMAScript/JavaScript code. Created by Nicholas Zakas in 2013 as a more configurable successor to JSLint and JSHint, its defining design choice is that the linter core knows almost nothing: it parses source into an AST, walks it, and dispatches node visits to rules. Every single rule — including the built-ins — is a plugin operating on that tree[^1]. This made ESLint extensible in a way its predecessors were not, and is why the ecosystem (React, TypeScript, Vue, import-order, accessibility) standardized on it rather than forking the core.

The project is now governed under the OpenJS Foundation. It is one of the most-depended-upon packages in the npm registry: nearly every non-trivial JavaScript/TypeScript project runs it in CI. With ~27.5k GitHub stars and ~5.2k forks it under-indexes on stars relative to its actual install base, which is the normal signature of infrastructure tooling that people configure once and forget.

The central tension in modern ESLint is scope. ESLint predates Prettier and TypeScript's own type-checker, and historically absorbed formatting rules (indentation, quotes, semicolons) and some type-adjacent checks. The team has since deliberately narrowed the mission: formatting rules were frozen and are being deprecated in favor of Prettier or dedicated formatters[^2], and stylistic rules were spun out to the community `@stylistic` project. ESLint's job is code *correctness and quality*, not layout.

## Getting Started

```shell
npm init @eslint/config@latest
```

This scaffolds an `eslint.config.js` (the flat config format, default since v9). A minimal config:

```js
// eslint.config.js
import js from "@eslint/js";
import { defineConfig } from "eslint/config";

export default defineConfig([
	js.configs.recommended,
	{
		files: ["**/*.js", "**/*.mjs", "**/*.cjs"],
		rules: {
			"prefer-const": "warn",
			"no-unused-vars": "error",
			"no-constant-binary-expression": "error",
		},
	},
]);
```

```shell
npx eslint .          # lint the project
npx eslint . --fix    # apply auto-fixable rules
```

Rule severities are `"off"`/`0`, `"warn"`/`1` (no effect on exit code), and `"error"`/`2` (exit code 1). Requires Node.js `^20.19.0`, `^22.13.0`, or `>=24`.

## Architecture / How It Works

The pipeline is: **parse → traverse → rule visitors → report → (optional) fix**.

1. **Parsing.** Source is parsed to an ESTree-compatible AST by [Espree](https://github.com/eslint/js) (a wrapper over Acorn) by default. The parser is swappable — `@typescript-eslint/parser`, `@babel/eslint-parser`, and `vue-eslint-parser` replace it to produce ASTs for supersets of JavaScript. ESLint's core rules only understand standard ESTree, which is why TypeScript-aware rules live in a separate plugin, not core.
2. **Traversal.** The AST is walked; on each node ESLint calls every rule that registered a visitor for that node type (e.g. `CallExpression`, `VariableDeclaration`). Rules also query **scope analysis** (via `eslint-scope`) to reason about variable bindings, and **source code / token** APIs for whitespace and comments.
3. **Reporting.** A rule calls `context.report({ node, messageId, fix })`. Fixes are described as text range replacements, not applied immediately.
4. **Fixing.** After all rules run, ESLint applies non-overlapping fixes and re-lints in a loop (up to 10 passes) until the output stabilizes.

The **flat config** system (`eslint.config.js`, default in v9, opt-in from v8.21) replaced the older `.eslintrc` + `extends` cascade. Config is now a plain array of objects evaluated in order, each optionally scoped by `files`/`ignores` globs. This removed the implicit resolution magic of `.eslintrc` (extends chains, `env` presets, plugin-name string resolution) in exchange for explicit imports — a cleaner mental model but a hard migration for large configs and shareable-config authors.

ESLint ships as a CommonJS package. Because of this it cannot freely `require()` ESM-only dependencies (a top-level `await` in a dependency would break it), so it constrains itself to `require(esm)` for its own zero-dependency ESM packages and dynamic `import()` for external ESM used only in async paths[^3].

## Production Notes

**Flat config migration is the dominant upgrade pain.** The v8→v9 jump makes `eslint.config.js` the default and removes built-in support for `.eslintrc`. Plugins had to publish flat-config-compatible entry points, and shareable configs written as `extends` strings do not port mechanically. Many teams stayed on v8 for months waiting for their plugin stack to catch up. The `@eslint/eslintrc` compat package can bridge old configs, but it is a stopgap, not a destination.

**Type-aware linting is slow and separate.** ESLint core does no type checking. `@typescript-eslint` rules that need type information (`no-floating-promises`, `no-unsafe-*`) invoke the TypeScript compiler behind the scenes; enabling them can multiply lint time several-fold on large repos. The `projectService` option and typed-vs-untyped rule splits exist specifically to manage this cost.

**Formatting rules are deprecated — don't fight Prettier.** The built-in stylistic/layout rules (`indent`, `quotes`, `semi`, etc.) are frozen and moved to `@stylistic/eslint-plugin`. Running ESLint formatting rules alongside Prettier produces conflicting auto-fixes. The current guidance is: Prettier (or another formatter) owns layout; ESLint owns logic[^2].

**`--fix` is not always safe.** Fixes are categorized as "fixable" (safe) and "suggestions" (require explicit opt-in) precisely because some transforms can change behavior. Autofix runs iteratively and can interact badly with overlapping rules from multiple plugins; review autofix diffs in CI-critical paths.

**Performance levers.** Use `--cache` to skip unchanged files between runs. Flat config's explicit `ignores` is faster than deep `.eslintignore` scanning. The `TIMING=1` env var prints per-rule time, which is the standard way to find the one expensive custom or type-aware rule dragging down a run.

**Version support is narrow.** The team supports the current major plus six months of limited (security/critical only) support for the previous major[^4]. Older majors go to commercial partners (Tidelift, HeroDevs). Plan major upgrades on that cadence rather than deferring indefinitely.

## When to Use / When Not

**Use when:**
- You want enforceable, per-rule-configurable code-quality checks on JS/TS/JSX in CI.
- You need custom lint rules encoding project-specific invariants — the plugin API is the reason to pick ESLint over a fixed-ruleset tool.
- You rely on the ecosystem: React hooks rules, `import/order`, `jsx-a11y`, framework plugins.

**Avoid (or supplement) when:**
- You only need formatting — use Prettier/dprint directly; ESLint is the wrong layer.
- You want a zero-config, fast, all-in-one Rust toolchain and can accept a smaller rule set — see Biome/Oxlint.
- You need whole-program type errors — that is `tsc`'s job; ESLint's type-aware rules are a complement, not a replacement.

## Alternatives

- biomejs/biome — Rust linter+formatter, single fast binary, no plugins yet; use when speed and a unified toolchain matter more than ESLint's rule breadth.
- oxc-project/oxc (Oxlint) — very fast Rust linter aiming at ESLint-rule parity; use as a pre-filter or when lint time dominates CI.
- prettier/prettier — a formatter, not a linter; use *alongside* ESLint for layout, not instead of it.
- typescript-eslint/typescript-eslint — the parser+plugin that makes ESLint understand TypeScript; not an alternative but a required companion for TS.
- microsoft/TypeScript (`tsc`) — use for type correctness; ESLint for stylistic and logical patterns `tsc` doesn't cover.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.2 | 2013-07 | First npm publish by Nicholas Zakas[^1]. |
| 1.0 | 2015-07 | First stable release. |
| 2.0 | 2016-02 | Config cascade, plugin system maturation. |
| 4.0 | 2017-06 | AST/autofix changes, shareable config improvements. |
| 6.0 | 2019-06 | Node engine bump, config resolution changes. |
| 8.0 | 2021-10 | ESM config support, flat config groundwork. |
| 8.21 | 2022-08 | Flat config available (opt-in via `eslint.config.js`)[^5]. |
| 9.0 | 2024-04 | Flat config becomes default; `.eslintrc` no longer default; formatting rules frozen[^2]. |

## References

[^1]: Nicholas C. Zakas, "Introducing ESLint" — 2013-07-30. https://humanwhocodes.com/blog/2013/07/16/introducing-eslint/
[^2]: ESLint blog, "Deprecation of formatting rules" — 2023-10. https://eslint.org/blog/2023/10/deprecating-formatting-rules/
[^3]: ESLint README, "ESM Dependencies". https://github.com/eslint/eslint#esm-dependencies
[^4]: ESLint, "Version Support". https://eslint.org/version-support
[^5]: ESLint blog, "Flat config rollout plans" — 2022-08. https://eslint.org/blog/2022/08/new-config-system-part-2/

## Tags

javascript, typescript, linter, static-analysis, code-quality, ast, eslint, ecmascript, developer-tools, nodejs, plugin-architecture
