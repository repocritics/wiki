# unjs/pathe

> A drop-in replacement for Node.js's `path` module that always normalizes to POSIX-style forward slashes and runs with no Node.js dependency.

[GitHub repo](https://github.com/unjs/pathe) ·
[npm](https://www.npmjs.com/package/pathe) ·
[License: MIT](https://github.com/unjs/pathe/blob/main/LICENSE)

## Overview

pathe is a small library from the UnJS collective that mirrors the API of Node.js's built-in `path` module but removes its single most surprising behavior: platform-dependent separators. Node's `path` silently switches between POSIX (`/`) and Windows (`\`) semantics depending on the OS the process runs on, which means the same code produces different strings on a developer's Mac and a Windows CI runner. pathe eliminates that branch — every operation normalizes to forward slashes on every platform[^1].

The practical audience is tooling authors. pathe is a transitive dependency across much of the modern JavaScript build ecosystem (Vite, Vitest, Nuxt, and many other UnJS-adjacent packages depend on it), so its GitHub star count badly understates its reach — it is one of the most-installed packages in the UnJS org by npm download volume, ordered magnitudes above what 583 stars suggests. If you have a `node_modules` with a bundler in it, you almost certainly already ship pathe.

The defining tradeoff is deliberate incorrectness relative to Windows. pathe does not try to be `path.win32`; it is closer to always being `path.posix` with backslash inputs coerced to slashes. That is exactly what you want for glob patterns, virtual filesystems, config path comparison, and browser code — and exactly what you do not want if you need a real Windows path to hand to a native API that rejects forward slashes. The `delimiter` constant is the one intentional exception: it stays `;` on Windows[^1].

## Getting Started

```bash
npm i pathe
```

```js
// ESM / TypeScript — identical named exports to node:path
import { resolve, join, normalize, matchesGlob } from "pathe";

// CommonJS
const { resolve, join } = require("pathe");

// On Windows, node:path would return "C:\\a\\b\\c";
// pathe always returns "C:/a/b/c"
resolve("C:\\a", "b", "c"); // => "C:/a/b/c"
join("foo\\bar", "..", "baz"); // => "foo/baz"
matchesGlob("src/index.ts", "src/**/*.ts"); // => true
```

Extra helpers that do not exist in Node's `path` live under a subpath:

```js
import {
  filename,
  normalizeAliases,
  resolveAlias,
  reverseResolveAlias,
} from "pathe/utils";
```

## Architecture / How It Works

pathe is a pure-TypeScript reimplementation of the `path` algorithms, not a wrapper around `node:path`. That is the load-bearing design decision: because it does not import Node's module, it carries no Node built-in dependency and runs unchanged in browsers, edge runtimes, and Web Workers where `node:path` is unavailable or polyfilled[^1].

The core exports (`resolve`, `join`, `normalize`, `dirname`, `basename`, `extname`, `relative`, `isAbsolute`, `parse`, `format`, `sep`, `delimiter`, plus `matchesGlob`) match Node's names and signatures. Internally each function runs Node's POSIX logic and then normalizes separators, so a backslash in the input is treated as a path separator regardless of platform. Drive letters (`C:`) and UNC-style prefixes are recognized and preserved; the output just uses `/` between segments.

`matchesGlob` — the newer addition mirroring Node's experimental `path.matchesGlob` — is powered by the external `zeptomatch` matcher rather than a hand-rolled implementation[^1]. This is the one place pathe pulls in a runtime dependency of consequence.

The `pathe/utils` subpath is a separate concern: alias resolution. `normalizeAliases`, `resolveAlias`, and `reverseResolveAlias` implement the path-alias mapping logic used by build tools (the `@/` → `src/` style rewrites in tsconfig `paths` / Vite `resolve.alias`), and `filename` extracts a basename without extension. These are not part of the Node `path` surface and exist because the same tooling authors who need normalized paths also repeatedly need alias math.

## Production Notes

**It is not a Windows-path library.** The most common misuse is reaching for pathe when you actually need a native Windows path. Most Windows APIs and cmd builtins accept forward slashes, but some do not (certain `.bat` invocations, tools that string-split on `\`, registry-style paths). If you must emit a real backslash path, use `node:path` (`path.win32`) at that boundary and keep pathe for everything internal.

**`delimiter` is the deliberate inconsistency.** Every separator normalizes, but the PATH-list delimiter constant remains platform-specific (`;` on Windows, `:` elsewhere). Code that assumes "pathe normalizes everything" can be surprised here. This is intentional — the delimiter is about environment variables, not filesystem paths.

**Comparisons and cache keys get simpler.** The real payoff in production is that path strings become stable identifiers. Using pathe-normalized paths as `Map` keys, cache keys, or in snapshot assertions removes an entire category of "passes on Linux, fails on Windows CI" bugs. This is why test runners and bundlers adopt it.

**Bundle and dependency cost.** For browser or edge bundles, pathe is small and tree-shakeable, but pulling in `matchesGlob` transitively pulls in `zeptomatch`; if you only use `join`/`resolve`, tree-shaking should drop it, but verify in your bundle analysis if size is critical.

**Migration is usually a find-and-replace.** Because the export surface matches `node:path`, adopting pathe is typically changing `from "node:path"` to `from "pathe"`. The behavior change to watch for is any code path that was relying on backslash output on Windows — that silently changes.

## When to Use / When Not

**Use when:**
- You are writing cross-platform tooling (bundlers, test runners, linters, codegen) and want path strings that are identical on every OS.
- You need `path`-style utilities in a browser, edge runtime, or Worker where `node:path` is not available.
- You compare, hash, or snapshot paths and want to kill Windows/POSIX divergence bugs.
- You build glob patterns from filesystem paths and need `/` separators.

**Avoid when:**
- You must produce genuine Windows backslash paths for a native API — use `node:path`'s `win32` variant at that seam.
- You only need to convert separators once and nothing else — a single-purpose helper like `slash` is lighter.
- You want byte-for-byte parity with `node:path` output on Windows, including backslashes — pathe intentionally does not provide that.

## Alternatives

- nodejs/node — the built-in `node:path`; use it when you need real platform-native semantics (true Windows backslashes) or want zero dependencies in a Node-only process.
- anodynos/upath — the older POSIX-normalizing `path` wrapper pathe was built to replace; use it only for legacy CommonJS Node-only codebases already depending on it. pathe is the ESM/TypeScript, no-Node-dependency successor.
- sindresorhus/slash — a one-function utility that flips backslashes to forward slashes; use it when separator conversion is literally all you need and you don't want the full `path` API.
- jinder/path (path-browserify) — a browser polyfill of Node's `path` that preserves platform semantics; use it when you want Node `path` behavior in the browser without pathe's always-normalize stance.
- fabiospampinato/zeptomatch — the glob matcher pathe uses under `matchesGlob`; use it directly when you only need glob matching and not path utilities.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-09 | First published in the UnJS org as a normalized `path` drop-in.[^2] |
| 1.x | 2022 | Stable core `path` API parity; broad adoption as a transitive dep across Vite/Vitest/Nuxt tooling. |
| 2.0 | 2025-02 | Major release adding `matchesGlob` (via `zeptomatch`) and refreshed internals.[^3] |

## References

[^1]: pathe README — "Universal filesystem path utils", rationale, usage, and `pathe/utils` subpath. https://github.com/unjs/pathe#readme
[^2]: GitHub repository metadata for unjs/pathe (created 2021-09-22). https://github.com/unjs/pathe
[^3]: pathe releases and changelog on GitHub / npm. https://github.com/unjs/pathe/releases

## Tags

typescript, javascript, nodejs, path, filesystem, cross-platform, posix, esm, unjs, tooling, browser
