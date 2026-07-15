# vercel/ncc

> Zero-config CLI that bundles a Node.js app and all its dependencies into a single file, webpack under the hood.

[GitHub repo](https://github.com/vercel/ncc) ·
[npm: @vercel/ncc](https://npmjs.com/@vercel/ncc) ·
[License: MIT](https://github.com/vercel/ncc/blob/main/LICENSE)

## Overview

`ncc` (originally "node-cc", the Node compact compiler) compiles a Node.js
entry point plus its entire `require`/`import` graph into one JavaScript file,
"gcc-style"[^1]. The pitch is a compiled-language experience for server code:
point it at `input.js`, get `dist/index.js` with no `node_modules` shipped.
Its stated design goals are zero configuration, built-in TypeScript, and
Node-only input/output — it is explicitly not a general-purpose web bundler[^1].

Under the hood `ncc` is a thin, opinionated wrapper around webpack plus
`@vercel/webpack-asset-relocator-loader`, which is what lets it trace and
relocate native `.node` addons and statically analyzable asset reads. That
choice is the project's defining tension: it inherits webpack's correctness for
gnarly dependency graphs, but also webpack's weight, its static-analysis blind
spots, and a build-time cost that lighter bundlers avoid.

The dominant real-world use is GitHub Actions: the official `typescript-action`
template compiles an Action into a single committed `dist/index.js` with `ncc`
so it runs without an install step[^2]. That one use case drives most of the
traffic, issues, and reason the tool still matters despite low release activity.

## Getting Started

```bash
npm i -g @vercel/ncc
```

```bash
# Bundle input.js and everything it requires into dist/index.js
ncc build input.js -o dist

# Minify, and emit a source map
ncc build src/index.ts -o dist -m -s

# Build-and-run in a temp dir with source-map support (debugging)
ncc run input.ts
```

```js
// Programmatic API
const ncc = require('@vercel/ncc');

ncc('/path/to/input.ts', {
  minify: true,
  sourceMap: true,
  externals: ['aws-sdk'],   // leave as bare require() in output
  target: 'es2015',
}).then(({ code, map, assets }) => {
  // assets: { [filename]: { source, permissions, symlinks } }
});
```

TypeScript needs a `tsconfig.json`; if `typescript` is in `devDependencies`
that version is used, otherwise ncc's bundled one[^1].

## Architecture / How It Works

`ncc` orchestrates a single-pass webpack build with a fixed config the user
mostly cannot see or override. The pipeline:

- **Module graph** — webpack resolves the `require`/`import` graph from the
  entry and inlines everything into one chunk. `--external <mod>` (or the
  `externals` option) leaves a module as a runtime `require`, which is how you
  keep unbundleable or host-provided packages (e.g. `aws-sdk` on Lambda) out of
  the bundle.
- **Asset relocation** — `@vercel/webpack-asset-relocator-loader` statically
  evaluates expressions like `fs.readFileSync(__dirname + '/x')` and
  `require('bindings')(...)` to discover files and native addons that are read
  at runtime, then copies them next to the output and rewrites the path[^3].
  This is the mechanism behind "supports binary addons and dynamic requires."
- **TypeScript** — handled through `ts-loader`. `--transpile-only` (`-t`) skips
  type-checking for faster builds, matching `ts-loader`'s `transpileOnly`.
- **Caching** — builds populate an on-disk cache (`ncc cache dir|size|clean`);
  `-C`/`--no-cache` skips it.
- **Output shape** — CommonJS by default. If the input resolves inside a
  `"type": "module"` package boundary, or uses `.mjs`/`.cjs`, the output
  extension and format follow accordingly[^1].

Because the config is webpack's, ncc's correctness ceiling is webpack's: what
webpack can bundle, ncc can bundle; what defeats webpack's static analysis
(computed `require`, runtime-constructed paths) defeats ncc too. The relocator
narrows that gap for the common asset patterns but does not close it.

## Production Notes

- **Dynamic, non-static requires are the primary footgun.** The relocator only
  finds assets it can evaluate statically. `require(variable)`,
  `require(path.join(dir, name))`, and runtime-globbed plugin loaders silently
  drop files from the bundle — the build succeeds and fails at runtime. This is
  the single most common ncc bug report, and it is inherent to the approach,
  not a fixable defect[^1].
- **Native addons are hit-or-miss by package.** Modules using `node-gyp`
  outputs, `node-pre-gyp`, or `bindings` sometimes need `--external` or manual
  asset handling; the repo keeps `package-support.md` for per-package
  workarounds[^1].
- **`dist/` is meant to be committed for GitHub Actions.** Run `ncc build`,
  commit `dist/index.js`, and gate CI on it being current — forgetting to
  rebuild after a source change ships stale code, so teams add a "check dist"
  CI step[^2].
- **Build cost is webpack cost.** For a GitHub Action this is fine; as a general
  server-bundling step in a hot inner loop, ncc is noticeably slower than
  esbuild-class tools. `--transpile-only` helps by dropping type-checking.
- **Maintenance is low-activity.** ncc has sat at a `0.x` version for its entire
  life and open issues have accumulated (200+), with releases dominated by
  dependency bumps rather than feature work. It is stable and widely depended
  upon, but treat it as done rather than evolving — do not expect fixes for
  edge-case bundling failures to land quickly.
- **Source maps add weight.** Enabling `sourceMap` auto-injects
  `source-map-support` into the output (~32 kB); `--no-source-map-register`
  opts out if you register it yourself[^1].

## When to Use / When Not

**Use when:**
- You are building a GitHub Action and want the standard single-file `dist/`.
- You want to publish or deploy a Node service without shipping `node_modules`,
  and your dependency graph is mostly statically analyzable.
- You want TypeScript-in, single-file-out with no bundler config to maintain.

**Avoid when:**
- Your app loads modules or assets by computed/dynamic paths — the static
  relocator will miss them.
- You need fast, repeated bundling in a dev loop — esbuild/tsup are far quicker.
- You want an actual self-contained executable (not just one `.js` needing a
  Node runtime) — use a compiler that embeds Node.
- You need an actively developed bundler with responsive maintenance.

## Alternatives

- evanw/esbuild — much faster Go-based bundler; `--bundle --platform=node` covers
  the same "one file" goal when you don't need webpack's asset relocation.
- egoist/tsup — esbuild wrapper aimed at libraries; use when you want dual
  CJS/ESM output and declaration files rather than a single app bundle.
- vercel/nft (`@vercel/nft`) — traces the file set a Node app needs instead of
  bundling into one file; use when you want to ship a minimal `node_modules`
  subset with native modules intact rather than inline everything.
- yao-pkg/pkg (successor to vercel/pkg) — use when you need an actual standalone
  executable with an embedded Node runtime, not a script.
- rollup/rollup — use when you want fine-grained control over output format and
  tree-shaking for a library rather than zero-config app bundling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2018-11 | Initial release; webpack-based single-file Node compiler[^1]. |
| 0.x | 2019–2021 | TypeScript via ts-loader, asset relocation, `run` command, ESM output for `"type":"module"` boundaries. |
| 0.38.x | 2023–2024 | Late maintenance line; primarily dependency and webpack updates. |
| — | 2026-07 | Repo still receiving commits (dependency maintenance), no major version[^4]. |

## References

[^1]: `@vercel/ncc` README — motivation, design goals, CLI options, caveats. https://github.com/vercel/ncc#readme
[^2]: GitHub `typescript-action` template — uses ncc to compile Actions into a committed single-file `dist/`. https://github.com/actions/typescript-action
[^3]: `@vercel/webpack-asset-relocator-loader` — static asset/native-addon relocation used by ncc. https://github.com/vercel/webpack-asset-relocator-loader
[^4]: Repo metadata (stars, forks, last push) via GitHub API, fetched 2026-07. https://github.com/vercel/ncc

## Tags

javascript, nodejs, bundler, webpack, cli, typescript, github-actions, build-tool, single-file, zero-config
