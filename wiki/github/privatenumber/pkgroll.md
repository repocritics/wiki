# privatenumber/pkgroll

> Zero-config Rollup bundler for npm packages — the exports map *is* the build config.

[GitHub repo](https://github.com/privatenumber/pkgroll) ·
[npm](https://npm.im/pkgroll) ·
[License: MIT](https://github.com/privatenumber/pkgroll/blob/master/LICENSE)

## Overview

pkgroll is a bundler for Node.js library authors, written by Hiroki Osame
(privatenumber, also the author of `tsx`). First released in March 2022[^1], it
inverts the usual build-tool relationship: there is no `pkgroll.config.js` and
no plugin API. Instead it reads the fields Node.js itself uses for package
resolution — `main`, `module`, `types`, `bin`, `exports`, `imports` — maps
declared `./dist/*` outputs back to `./src/*` sources, and produces ESM,
CommonJS, and bundled `.d.ts` outputs that match what you declared[^2]. Rollup
does the module graph, tree-shaking, and CJS interop; esbuild handles
TypeScript transforms and minification.

The defining tradeoff is the config inversion itself. Because the exports map
is the single source of truth, pkgroll structurally prevents the most common
npm publishing failure — declaring entry points that don't match what the
build actually emits. The cost is that anything off the paved road (CSS, code
generation, custom Rollup plugins, per-entry tsconfigs) has no escape hatch
beyond a modest set of CLI flags. pkgroll is deliberately a convention, not a
toolkit.

At ~1.6k stars it is far smaller than tsup, but actively maintained — 90+
releases since 2022 on a semantic-release cadence, pushes as recent as July
2026 — and occupies a distinct niche: the strictest interpretation of "the
package manifest is the spec."

## Getting Started

```sh
npm install --save-dev pkgroll
```

Declare outputs in `package.json`; sources live in `src/`:

```json
{
    "name": "my-package",
    "type": "module",
    "exports": {
        "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" },
        "import": { "types": "./dist/index.d.mts", "default": "./dist/index.mjs" }
    },
    "bin": "./dist/cli.js",
    "scripts": { "build": "pkgroll" }
}
```

```sh
npm run build   # src/index.ts → dist/index.{mjs,cjs,d.mts,d.cts}; src/cli.ts → executable bin
```

## Architecture / How It Works

- **Entry-point derivation.** Every `./dist/` path found in `main`, `module`,
  `types`, `bin`, `exports` (including wildcard subpath patterns like
  `"./utils/*": "./dist/utils/*.mjs"`), and `imports` becomes a bundle entry,
  resolved from `./src/` (configurable via `--srcdist`).
- **Format inference mirrors Node.js.** Output format per entry follows the
  same lookup Node uses: `.cjs`/`.mjs` are explicit, `.js` falls back to
  `package.json#type`, `exports.require` forces CJS, `types` fields produce
  declaration bundles[^3]. You never state formats twice.
- **Rollup core, esbuild transforms.** Rollup is chosen specifically for its
  CJS output quality: emitted CommonJS keeps named exports statically
  analyzable by `cjs-module-lexer`, so Node's ESM loader can named-import from
  the CJS build[^2]. esbuild does TS-to-JS and minification for speed.
- **Externalization by dependency type.** `dependencies`, `peerDependencies`,
  and `optionalDependencies` are externalized; `devDependencies` are bundled
  in (their types tree-shaken into the `.d.ts` bundle). Where a dependency
  lands in the manifest *is* the bundling decision — there is no `external`
  flag.
- **ESM/CJS shims.** `require()` in source compiled to ESM is shimmed with
  `createRequire(import.meta.url)`. `bin` entries get a hashbang (respecting a
  custom one in source, e.g. `#!/usr/bin/env bun`) and `chmod 0755`.
- **Aliases and attributes.** Aliases come from `package.json#imports` or
  `tsconfig.json#paths`; import attributes (`with { type: 'text' }`/`'bytes'`)
  inline files at build time; imported `.node` addons are copied to
  `dist/natives/`.

## Production Notes

- **The `devDependencies` = bundled convention surprises people.** Native
  modules (`fsevents`, anything using `bindings`/`node-pre-gyp`) in
  `devDependencies` break the build or the bundle; the fix is moving them to
  `dependencies` to externalize[^3]. Bundling third-party code also creates
  license obligations — `--license` generates a bundled-dependencies file.
- **Build targets default to the *running* Node version.** Without an explicit
  `--target`, output syntax depends on which Node executed the build — dev and
  CI can emit different code. Pin `--target` for reproducible publishes[^3].
- **Declaration bundling chokes on non-JS entries.** If `exports` references
  CSS or other assets, `.d.ts` bundling errors; the `--packagejson` glob
  filter exists largely to work around this by building subsets of entries.
- **Correct manifests are still your job.** pkgroll faithfully builds whatever
  the exports map says — including a wrong one (`types` condition ordering,
  `.d.cts` vs `.d.mts` mismatches). Pair with `publint` and
  `@arethetypeswrong/cli` in CI.
- **No plugin escape hatch means a migration cliff.** The day you need a
  Rollup plugin, you rewrite the build in tsup/tsdown/unbuild rather than
  extend pkgroll. Budget for that if the package may grow assets or codegen.
- **Single-maintainer project; sponsor-gated development repo.** Release
  cadence is healthy, but development discussion happens in a separate repo
  accessible to GitHub sponsors[^1] — bus factor and roadmap visibility are
  what you'd expect from a one-person tool. Node.js ≥ 18 required since
  v2[^4]; TypeScript is an optional peer at `^4.1 || ^5.0`.

## When to Use / When Not

**Use when:**
- You publish TypeScript/ESM libraries to npm and want dual ESM+CJS+`.d.ts`
  output with zero build config.
- You want the exports map, not a bundler config, as the single source of
  truth — structural protection against manifest/build drift.
- You ship CLIs (`bin` handling, hashbangs) alongside library entry points.

**Avoid when:**
- You need bundler plugins, CSS/asset pipelines, or codegen — there is no
  extension point; use tsup, tsdown, or unbuild.
- You bundle for browsers or apps rather than npm distribution — this is a
  package bundler, not an app bundler.
- The team wants explicit, reviewable build config; the implicit dist→src
  mapping can read as magic.

## Alternatives

- egoist/tsup — esbuild-based with a config file and plugin hooks; use it when
  you need customization beyond what `package.json` can express.
- rolldown/tsdown — Rust (Rolldown) based library bundler positioned as the
  tsup successor; use it for faster builds within the Vite-adjacent toolchain.
- unjs/unbuild — Rollup-based with explicit config and a passive "stub" dev
  mode; the convention across the UnJS/Nuxt ecosystem.
- evanw/esbuild — script it directly when you want maximum control and don't
  need `.d.ts` bundling or Node-exact CJS interop.
- vitejs/vite — library mode, when the package lives inside a Vite app anyway.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2022-03-31 | Initial release, same day the repo was created[^1]. |
| 1.3–1.11 | 2022–2023 | Steady buildout: aliases, export conditions, watch mode. |
| 2.0.0 | 2023-10-02 | Breaking: Node.js ≥ 18; TypeScript & Rollup upgrades[^4]. |
| 2.9–2.15 | 2025 | Rapid minor cadence: wildcard exports, subpath imports, license extraction. |
| 2.27.1 | 2026-05-27 | Latest release at time of writing; repo pushed as recently as 2026-07. |

## References

[^1]: pkgroll repository, created 2022-03-31. https://github.com/privatenumber/pkgroll
[^2]: pkgroll README — "Why bundle with Rollup?" (CJS named-export analyzability via cjs-module-lexer). https://github.com/privatenumber/pkgroll#faq
[^3]: pkgroll README — Usage: output formats, dependency externalization, target defaults. https://github.com/privatenumber/pkgroll#usage
[^4]: pkgroll v2.0.0 release notes — 2023-10-02, "Minimum Node.js support increased to v18 and above". https://github.com/privatenumber/pkgroll/releases/tag/v2.0.0

## Tags

typescript, nodejs, bundler, rollup, esbuild, zero-config, npm-packaging, dual-package, dts-bundling, build-tool, cli-tooling
