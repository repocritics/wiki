# stylelint/stylelint

> A PostCSS-based CSS linter that catches errors and enforces conventions — and, since v15, deliberately no longer formats your code.

[GitHub repo](https://github.com/stylelint/stylelint) ·
[Official website](https://stylelint.io) ·
[License: MIT](https://github.com/stylelint/stylelint/blob/main/LICENSE)

## Overview

Stylelint is the de facto standard linter for CSS and CSS-like languages. It was created in 2014 by David Clark and Maxime Thirouin and is built on top of PostCSS: it parses stylesheets into an abstract syntax tree, then runs a set of rules that walk that tree and report problems[^1]. It ships with over 100 built-in rules and can be extended with plugins, shareable configs, and custom syntaxes to lint SCSS, Sass, Less, SugarSS, and styles embedded in HTML, Markdown, or CSS-in-JS template literals.

Its defining decision is what it chose to stop doing. Through v14, Stylelint carried a large body of *stylistic* rules (indentation, whitespace, newline placement) that overlapped with dedicated formatters. In v15 (2023) those ~76 rules were deprecated, and in v16 they were removed from core, with the project explicitly recommending Prettier for formatting and reserving Stylelint for correctness and convention checks[^2]. This is the central tension for anyone adopting it: Stylelint is now a *code-quality* tool, not a formatter, and expects a pretty-printer to sit beside it.

The second thing to understand up front is that Stylelint is deliberately unopinionated out of the box. With no config it reports almost nothing; the useful behavior lives in shareable configs (`stylelint-config-standard`, `stylelint-config-recommended`) and in the plugin ecosystem. It is a rule engine plus an ecosystem, not a batteries-included preset.

## Getting Started

```bash
npm install --save-dev stylelint stylelint-config-standard
```

```json
// .stylelintrc.json
{
  "extends": "stylelint-config-standard",
  "rules": {
    "declaration-no-important": true,
    "selector-max-id": 0
  }
}
```

```bash
# lint, then auto-fix what can be fixed safely
npx stylelint "**/*.css"
npx stylelint "**/*.css" --fix
```

For SCSS, swap in the SCSS toolchain, which pulls in the `postcss-scss` custom syntax and SCSS-aware rules:

```bash
npm install --save-dev stylelint-config-standard-scss
# extends: "stylelint-config-standard-scss"
```

## Architecture / How It Works

Stylelint is a thin orchestration layer over three PostCSS-adjacent concerns: **parsing**, **rules**, and **reporting**.

- **Parsing / custom syntaxes.** Stylelint never parses CSS itself. It hands source text to a PostCSS syntax that returns an AST. The default syntax is plain CSS; `customSyntax` swaps in `postcss-scss`, `postcss-less`, `postcss-sass`, `postcss-html`, or a user-supplied parser. This is why Stylelint can lint SCSS or `<style>` blocks in Vue/HTML: the language support is entirely in the syntax layer, and rules operate on whatever nodes the parser produced.
- **Rules.** Each rule is a function that walks the AST (`root.walkRules`, `walkDecls`, `walkAtRules`) and calls `report()` on violations. Rules are independent and stateless per lint run; there is no shared analysis pass, which keeps rules simple but means cross-rule information (e.g. a global symbol table) is not available. A subset of rules implement a `fix` path that mutates the AST in place.
- **Configuration cascade.** Config is resolved via cosmiconfig, so `.stylelintrc.*`, a `stylelint` key in `package.json`, or a JS/TS config file all work. `extends` merges shareable configs; `overrides` scope rules by file glob; `plugins` register additional rule namespaces (conventionally prefixed, e.g. `scss/`).
- **Reporting / formatters.** Results flow through pluggable formatters (`string`, `json`, `compact`, `github`, custom). Editor integrations and the `stylelint-vscode`/`vscode-stylelint` bridge consume the Node API rather than the CLI.

Stylelint exposes three entry points that share this core: the **CLI**, the **Node.js API** (`stylelint.lint(...)`), and a **PostCSS plugin** form for teams already running a PostCSS pipeline. The plugin ecosystem (`stylelint-scss`, `stylelint-order`, `stylelint-a11y`, and hundreds more) is where most real-world rule power lives; core intentionally stays language-neutral.

## Production Notes

- **It is not a formatter — stop configuring it like one.** Post-v16, whitespace/indentation rules are gone from core. Teams upgrading from v14 that relied on those rules must move formatting to Prettier (or add the community `@stylistic/stylelint-plugin`, which revived the removed rules)[^2]. Configs that reference removed rules will error on startup, not warn.
- **The v16 ESM migration bit hard.** v16 shipped as pure ESM[^3]. Projects with CommonJS tooling, older Node, or `require()`-based custom configs/plugins needed rewrites or dynamic import. Some plugins lagged the ESM cut for months. If you are pinned to an old plugin, check its Stylelint peer-dep range before upgrading.
- **Performance scales with rule count and glob size, not just file count.** Every enabled rule re-walks the AST. Large monorepos linting `**/*.{css,scss}` with a heavy config feel slow; the standard mitigations are narrowing globs, `--cache` (persists results between runs), and running only in CI plus editor rather than pre-commit on the whole tree.
- **`--fix` is not universally safe to run unattended.** Only rules with a fixer participate, and a few fixers can change meaning in edge cases (e.g. shorthand rewrites). Run fixes through review, not as a silent commit hook, especially on SCSS where the syntax layer round-trips through `postcss-scss`.
- **Custom-syntax mismatches produce confusing errors.** Linting SCSS with the default (CSS) syntax reports spurious "unknown" errors on `@mixin`/`$vars`; the fix is always to set the right `customSyntax` (usually via the matching `-scss` config). Embedded-styles linting (`postcss-html`) has its own gaps around template interpolation.
- **Node support moves forward aggressively.** Major versions drop EOL Node lines. Confirm your Node version against the target major before upgrading rather than after a red CI run.

## When to Use / When Not

**Use when:**
- You want to catch CSS/SCSS errors and enforce conventions (naming patterns, allowed units, selector limits, modern color-function notation).
- You already run Prettier (or another formatter) and want a linter that complements it rather than fighting it.
- You need to lint non-plain-CSS: SCSS, Less, or styles embedded in HTML/Markdown/CSS-in-JS.
- You want an extensible rule engine and are willing to assemble a config from shareable presets and plugins.

**Avoid when:**
- You mainly want automatic formatting — that is Prettier's job now, not Stylelint's.
- You want zero-config, batteries-included linting with no preset to choose.
- Your stack is committed to a single all-in-one Rust toolchain (Biome) and you want CSS lint/format in one binary with no plugin layer.
- You are pinned to CommonJS-only tooling and cannot adopt the ESM-only v16+ line.

## Alternatives

- prettier/prettier — use instead for the *formatting* half; in practice run it alongside Stylelint, not as a replacement.
- biomejs/biome — use when you want CSS linting and formatting in one fast Rust binary and can accept a smaller, non-pluggable rule set.
- eslint/eslint — the JavaScript-side counterpart; pair with Stylelint rather than choosing between them.
- stylelint-scss (kristerkari/stylelint-scss) — not a competitor but the essential plugin when your codebase is SCSS-heavy.
- csstree/csstree — use when you need low-level CSS parsing/validation to build your own tooling rather than an off-the-shelf linter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 14.0.0 | 2021-09 | Node support bump; config and rule cleanups[^4]. |
| 15.0.0 | 2023-02 | Deprecated ~76 stylistic/formatting rules; recommends Prettier alongside[^2]. |
| 16.0.0 | 2023-12 | Pure ESM; removed the deprecated stylistic rules from core[^3]. |
| 17.0.0 | 2026 | Latest major; see the to-17 migration guide[^5]. |

## References

[^1]: Stylelint documentation and repository — "A mighty CSS linter that helps you avoid errors and enforce conventions." https://stylelint.io
[^2]: Stylelint 15.0.0 announcement and "Migrating to 15.0.0" — deprecation of stylistic rules and Prettier recommendation. https://stylelint.io/migration-guide/to-15
[^3]: Stylelint "Migrating to 16.0.0" — pure ESM migration and removal of deprecated stylistic rules. https://stylelint.io/migration-guide/to-16
[^4]: Stylelint "Migrating to 14.0.0". https://stylelint.io/migration-guide/to-14
[^5]: Stylelint "Migrating to 17.0.0". https://stylelint.io/migration-guide/to-17

## Tags

javascript, css, linter, postcss, scss, static-analysis, code-quality, developer-tools, cli, node
