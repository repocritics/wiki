# prettier/prettier

> An opinionated code formatter that reprints your code from its own AST, discarding the original styling.

[GitHub repo](https://github.com/prettier/prettier) ·
[Official website](https://prettier.io) ·
[License: MIT](https://github.com/prettier/prettier/blob/main/LICENSE)

## Overview

Prettier is a code formatter that parses source into an abstract syntax tree and re-prints it from scratch, throwing away almost all of the author's original whitespace and line breaks[^1]. It supports JavaScript, TypeScript, JSX, Flow, JSON, CSS/SCSS/Less, HTML, Vue, Angular templates, GraphQL, Markdown, and YAML out of the box, with a plugin API for additional languages. Since its 2017 debut by James Long it has become the default formatter for the JavaScript ecosystem — the "code style: prettier" badge and on-save editor integration are near-ubiquitous.

The defining design decision is that Prettier is *opinionated*: it deliberately exposes very few formatting options. The stated goal is to end style debates by removing the choices that produce them[^2]. Where linters like ESLint report a stylistic violation and ask you to fix it, Prettier simply rewrites the file. This is its central tension. Teams that want a specific house style (aligned assignments, custom brace placement, preserved manual line breaks) will find Prettier uncompromising by design; teams that want to stop arguing accept its output and move on. The project has repeatedly declined feature requests for new options on the grounds that each one reintroduces the debates the tool exists to eliminate.

Prettier is a formatter, not a linter: it enforces layout, not correctness or code-quality rules. The standard pairing is Prettier for formatting plus ESLint (with stylistic rules disabled) for bug-class linting.

## Getting Started

```bash
npm install --save-dev --save-exact prettier
```

Pinning the exact version (`--save-exact`) is standard practice because a patch release can change formatting output and cause spurious diffs across a team.

```bash
# Rewrite all files in place
npx prettier . --write

# Check formatting without writing (CI gate)
npx prettier . --check
```

A minimal `.prettierrc.json`:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 80
}
```

Programmatic use (note: the API is async-only since v3):

```js
import prettier from "prettier";

const formatted = await prettier.format("const x=1", { parser: "babel" });
// => "const x = 1;\n"
```

## Architecture / How It Works

Prettier runs a three-stage pipeline: **parse → build an intermediate document → print**.

1. **Parse.** The selected parser converts source into an AST. Prettier does not own most of these parsers — it delegates to `@babel/parser`, the TypeScript compiler, `postcss` (CSS/Less/SCSS), `angular-html-parser`, `graphql`, `remark`/mdast (Markdown), and `yaml`, among others. Parser choice is what makes a "plugin" for a new language possible.

2. **Doc IR.** Each AST node is translated by a *printer* into an intermediate representation called a Doc — a small algebra of commands (`group`, `indent`, `line`, `softline`, `hardline`, `fill`, `ifBreak`)[^3]. This design descends from Philip Wadler's "A prettier printer" and Christopher Chedeau's adaptation of it. The Doc describes *intent* ("these arguments belong together; break them all or none") rather than concrete whitespace.

3. **Print.** A printer algorithm walks the Doc and decides, group by group, whether the content fits within `printWidth`. If a group fits on one line it stays flat; if not, its `line` breaks become newlines and the group is expanded. This is why changing one argument can reflow an entire call — the fit decision is made per group, not per character.

Because the output is generated purely from the AST, Prettier discards the input's own line breaks and indentation. The few things it *does* preserve are deliberate exceptions: blank lines between statements are collapsed to at most one, object literals keep their expanded/collapsed state based on whether the first line had a break, and `// prettier-ignore` comments exempt the following node.

The parsers, printers, and Doc commands are internal and co-evolve. Third-party plugins (Tailwind class sorting, import sorting, Astro, Svelte) hook into the printer and Doc API, which means a plugin is effectively coupled to Prettier's major version.

## Production Notes

**The v2 → v3 async break.** Prettier 3 rewrote the codebase to ESM and made `prettier.format()` return a Promise; the synchronous API was removed[^4]. Any tool that called Prettier synchronously (older editor plugins, some ESLint integrations, custom scripts) broke on upgrade. This was the single most disruptive migration in the project's history.

**Plugins are version-locked.** A plugin written against Prettier 2's Doc/printer internals will not necessarily load under 3, and vice-versa. Check plugin compatibility before a major upgrade; a lagging Tailwind or import-sort plugin can block the whole bump.

**Formatting output is not frozen across versions.** Even minor/patch releases occasionally change output. Without a pinned version, `prettier --check` in CI can start failing after an unrelated `npm install`. Pin the exact version and treat formatting bumps as their own commit.

**Performance.** Prettier is written in JavaScript and the TypeScript parser in particular is heavy. On large monorepos, formatting all files can take noticeably long. Mitigations: the `--cache` flag (added in 2.8) skips unchanged files, and running only on staged files via `lint-staged` in a pre-commit hook is the common pattern. Teams that find Prettier too slow increasingly evaluate the Rust-based Biome or dprint.

**Do not run Prettier through ESLint.** `eslint-plugin-prettier` (Prettier-as-a-lint-rule) works but is slow and noisy — every formatting difference becomes a lint error. The maintainers recommend running Prettier separately and using `eslint-config-prettier` only to *disable* ESLint's own stylistic rules so the two tools don't fight[^5].

**Config resolution.** Prettier uses cosmiconfig-style discovery (`.prettierrc`, `prettier` key in `package.json`, etc.) and reads `.prettierignore`. It also respects a subset of `.editorconfig`. Surprising diffs are often a config file resolved from an unexpected parent directory.

## When to Use / When Not

**Use when:**
- You want to end code-style debates and enforce one layout automatically on save / in CI.
- You work in the JS/TS/CSS/Markdown ecosystem it natively supports.
- You want editor and pre-commit integration that "just works" with near-zero configuration.

**Avoid when:**
- You need a specific house style Prettier refuses to produce (alignment, preserved manual breaks, custom brace style) — it is intentionally inflexible.
- Formatting throughput on a huge codebase is a bottleneck and a Rust formatter would pay off.
- You need linting for correctness/bug classes — Prettier only formats; pair it with a linter.
- You cannot tolerate output changing across versions in a context where you refuse to pin dependencies.

## Alternatives

- biomejs/biome — Rust formatter *and* linter, much faster, Prettier-compatible output for JS/TS/JSON; use when speed matters and its language coverage is enough.
- dprint/dprint — Rust, pluggable via WASM plugins, faster than Prettier; use when you want Prettier-like formatting with better performance and configurable plugins.
- eslint/eslint — a linter, not a formatter; use for correctness rules alongside Prettier (its own stylistic rules are deprecated).
- **rome** — the predecessor project that was archived and forked into Biome; use Biome instead.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017-01 | Initial public release by James Long; JS/Flow/JSX formatting[^1]. |
| 1.x | 2017–2019 | Added TypeScript, CSS/SCSS/Less, GraphQL, Markdown, YAML, HTML, Vue support. |
| 2.0 | 2020-03-21 | Default `arrowParens` → `always`, `trailingComma` → `es5`; consolidated CSS/HTML handling[^6]. |
| 2.8 | 2022-11 | `--cache` flag for incremental formatting. |
| 3.0 | 2023-07-05 | ESM rewrite, async-only API, `trailingComma` default → `all`, Markdown/plugin changes[^4]. |

## References

[^1]: James Long, "A Prettier JavaScript Formatter" — 2017-01. https://jlongster.com/A-Prettier-Formatter
[^2]: Prettier docs, "Option Philosophy". https://prettier.io/docs/option-philosophy
[^3]: Prettier docs, "Prettier's Intermediate Representation (Doc commands)". https://prettier.io/docs/plugins#the-doc-builder
[^4]: Prettier blog, "Prettier 3.0" — 2023-07-05. https://prettier.io/blog/2023/07/05/3.0.0.html
[^5]: Prettier docs, "Integrating with Linters". https://prettier.io/docs/integrating-with-linters
[^6]: Prettier blog, "Prettier 2.0 '2020'" — 2020-03-21. https://prettier.io/blog/2020/03/21/2.0.0.html

## Tags

code-formatter, javascript, typescript, css, linting, developer-tooling, ast, pretty-printer, cli, editor-integration, nodejs
