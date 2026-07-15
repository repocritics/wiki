# sindresorhus/globby

> User-friendly glob matching for Node.js — a convenience layer over fast-glob.

[GitHub repo](https://github.com/sindresorhus/globby) ·
[License: MIT](https://github.com/sindresorhus/globby/blob/main/license)

## Overview

globby is a Node.js library for finding files by glob patterns (`src/**/*.js`,
`!**/*.test.js`, and so on). It is not itself a matching engine: since its v8
rewrite it delegates the actual traversal and pattern matching to
[`fast-glob`](https://github.com/mrmlnc/fast-glob), and layers ergonomic
features on top[^1]. The pitch is "fast-glob, but with the sharp edges filed
off": multiple patterns in one call, negation-only pattern lists, automatic
directory expansion, `.gitignore` awareness, and `URL` support for `cwd`.

It is one of the most widely depended-on packages in the JavaScript tooling
ecosystem — build tools, linters, test runners, and CLI utilities pull it in
to resolve user-supplied file globs. That position, more than its raw
capability, is what makes it worth understanding: it is infrastructure that
sits under other infrastructure.

The defining tension is that globby is a thin convenience wrapper whose value
is entirely in its defaults and edge-case handling, not its core algorithm.
Almost anything it does you can do by calling fast-glob directly with more
configuration. What you buy is Sindre Sorhus's opinion of the sane behavior,
plus the `.gitignore` integration — the one feature fast-glob does not offer
and which is genuinely non-trivial to reimplement correctly.

## Getting Started

```sh
npm install globby
```

globby is ESM-only in current releases; there is no `require()` entry point.
Consume it from an ES module or via dynamic `import()`.[^2]

```js
import {globby} from 'globby';

// Multiple patterns, negation, and .gitignore respect in one call.
const paths = await globby(['src/**/*.js', '!src/**/*.test.js'], {
	gitignore: true,
});

console.log(paths);
//=> ['src/index.js', 'src/util.js', ...]
```

Synchronous and streaming variants exist for different call sites:

```js
import {globbySync, globbyStream} from 'globby';

const files = globbySync('**/*.md');

for await (const path of globbyStream('*.tmp')) {
	console.log(path); // memory-bounded over huge result sets
}
```

## Architecture / How It Works

globby is a small facade. A call flows roughly as: normalize patterns → split
positive and negative patterns → build one or more "glob tasks" (a
`{patterns, options}` pair) → hand each task to fast-glob → merge and filter
results. The `generateGlobTasks()` / `generateGlobTasksSync()` functions expose
that intermediate representation so other globbing tools can reuse globby's
pattern normalization without its execution.

The features that justify the wrapper:

- **Negation-only patterns.** `globby(['!*.json'])` implicitly prepends `**/*`
  so it means "everything except JSON", which is what most people expect but
  is not how fast-glob behaves on its own. This is controllable via
  `expandNegationOnlyPatterns`, and the prepended catch-all respects the `dot`
  option.
- **Directory expansion.** A bare `foo` is expanded to `foo/**/*` by default
  (`expandDirectories`), so you can pass directory names and get their
  contents.
- **`.gitignore` integration.** With `{gitignore: true}` globby reads
  `.gitignore` files from the cwd downward, and — when a `.git` directory is
  detected — parent `.gitignore` files up to the repo root, to reproduce Git's
  actual precedence rules. `globalGitignore` extends this to the user-level
  `core.excludesfile`. The generic `ignoreFiles` option applies the same
  machinery to `.eslintignore`, `.prettierignore`, `.babelignore`, etc.
- **Standalone predicates.** `isGitIgnored()` / `isIgnoredByIgnoreFiles()`
  return a `(path) => boolean` tester you can apply outside of a glob call.

The `.gitignore` handling is where the real complexity lives, and it interacts
directly with performance (below). Everything else is pattern bookkeeping.

## Production Notes

**gitignore has a fast path and a slow path, and you can accidentally take the
slow one.** When there are no negation patterns and no parent `.gitignore`
files, globby passes ignore patterns down to fast-glob so ignored directories
(like `node_modules`) are never traversed. The moment a negation pattern or a
parent `.gitignore` is involved, globby must traverse everything and filter
afterward to stay Git-compatible. On a large tree this is the difference
between skipping `node_modules` and walking all of it. For hot paths, prefer
specific ignore patterns without negations, or target a single file with
`ignoreFiles: '.gitignore'` instead of a recursive search.

**Patterns are POSIX-only.** Glob patterns must use forward slashes even on
Windows. A pattern containing a backslash silently fails to match rather than
erroring. Build patterns with `path.posix.join()`, and run literal filesystem
paths that contain glob metacharacters — `()`, `[]`, `{}`, backslashes —
through `convertPathToPattern()` before interpolating them into a pattern.
`C:/Program Files (x86)/*.txt` matches nothing until escaped.

**ESM-only is a migration cost.** Since the v12 line globby ships as ESM with
no CommonJS build. Projects still on `require()` are pinned to the v11.x line,
which is functional but no longer receiving features. The last major bump also
raised the Node.js floor (v14 targets Node 18+), so upgrading globby can force
both a module-system and a runtime-version decision at once.[^2]

**You inherit fast-glob's behavior and bugs.** Options not documented by globby
are passed through to fast-glob, so `onlyFiles`, `deep`, `followSymbolicLinks`,
`concurrency`, `dot`, and the `fs` adapter all come from there. When results
surprise you, the answer is often in fast-glob's docs, not globby's. If you
supply a custom `fs` adapter together with `gitignore`/`ignoreFiles`, it must
also implement `readFile`/`readFileSync` (and `stat` variants for
`globalGitignore`), because globby reads ignore files through the same adapter.

**Glob tasks carry a filesystem cache.** `generateGlobTasks()` output embeds a
cache; regenerate tasks per run rather than reusing them, or you will match
against stale directory state.

## When to Use / When Not

**Use when:**
- You resolve user-supplied file patterns and want Git-consistent ignore
  behavior for free.
- You need multiple patterns, negations, and directory expansion without
  wiring them yourself.
- You are building tooling (linter, formatter, bundler plugin) and want the
  same glob semantics users already expect from other tools.

**Avoid when:**
- You need to match against an in-memory list of strings rather than the
  filesystem — use micromatch or multimatch, which never touch disk.
- You are on CommonJS and cannot migrate to ESM — you are capped at v11.
- You want the absolute minimum dependency footprint or maximum throughput and
  do not need the `.gitignore` layer — call fast-glob (or tinyglobby) directly.
- You only ever match one simple pattern with no ignore files — the wrapper
  earns nothing over the engine.

## Alternatives

- mrmlnc/fast-glob — the engine globby wraps; use it directly when you want raw
  speed and none of the gitignore/negation-only conveniences.
- isaacs/node-glob — the original, canonical glob implementation; use when you
  want the reference matcher or a CommonJS-friendly dependency.
- micromatch/micromatch — match strings in memory, not the filesystem; use for
  filtering lists you already have.
- sindresorhus/multimatch — match a list of strings against multiple patterns;
  use when your inputs are paths-as-data rather than paths-on-disk.
- SuperchupuDev/tinyglobby — smaller, dependency-light modern glob; use when
  install size and cold-start matter more than globby's ignore-file features.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2014-06-21 | Initial release.[^3] |
| 8.0.0 | 2018 | Rewrite onto fast-glob as the matching core.[^1] |
| 11.x | 2020–2021 | Last CommonJS line; still the fallback for non-ESM projects.[^2] |
| 12.0.0 | 2021 | ESM-only; drops `require()` support.[^2] |
| 14.0.0 | 2024 | Raises Node.js floor to 18; ongoing fast-glob passthrough.[^2] |

## References

[^1]: globby README, "Based on `fast-glob` but adds a bunch of useful
features." https://github.com/sindresorhus/globby#readme
[^2]: globby release history (major-version notes on ESM and Node.js support).
https://github.com/sindresorhus/globby/releases
[^3]: Repository creation date via GitHub API (`created_at`
2014-06-21). https://github.com/sindresorhus/globby

## Tags

javascript, nodejs, glob, file-matching, gitignore, filesystem, cli-tooling, esm, fast-glob, pattern-matching
