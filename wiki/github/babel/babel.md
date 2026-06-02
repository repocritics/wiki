# babel/babel

The JavaScript compiler — converts modern (or even proposed) JavaScript into versions that older browsers / runtimes can execute. The substrate underlying webpack/Rollup/Vite + the JSX → JS transform.

## What it is

A toolchain for parsing JavaScript into an AST, transforming the AST (via plugins), and serializing it back to JavaScript. Used to compile JSX, TypeScript-stripped output, ES2015+ → ES5, decorators, JSX-flavored DSLs, and many language-proposal features into stable JS. The plugin architecture is the value — every preset is a curated bundle of transforms.

## Key features

- AST-based pipeline: `@babel/parser` → `@babel/traverse` → `@babel/generator`.
- Plugin / preset system — `@babel/preset-env`, `@babel/preset-react`, `@babel/preset-typescript`.
- Configurable target environments (browserslist) — only-transpile-what-you-need.
- Source maps, comment preservation, idempotent transforms.
- MIT-licensed.

## Tech stack

- TypeScript primary at the source level.
- Distributed as scoped npm packages (`@babel/*`).

## When to reach for it

- You're integrating with build tools that expect Babel (webpack, Rollup, older Vite configs).
- You need bleeding-edge ECMAScript proposals before they land in target environments.
- You're writing JS transforms as a plugin author.

## When *not* to reach for it

- You're starting fresh in 2026 — esbuild / swc / oxc are faster substitutes for most use cases.
- You're TypeScript-first — `tsc` or swc handle TS+JSX without Babel.
- You don't need any transforms — modern Node + browsers handle most ES features natively.

## Maturity signal

44k stars, 6k forks, MIT, actively maintained though pace has slowed as faster alternatives (swc, esbuild) took over typical use cases. 11+ years.

## Alternatives

- esbuild — fastest single-pass alternative for most cases.
- swc — Rust-implemented JS transpiler; Next.js's default.
- oxc — newer Rust-implemented compiler suite.
- TypeScript `tsc` — for TS-only projects.

## Tags

javascript, typescript, compiler, ast, babel, build-tool, frontend, mit-license
