# unjs/citty

> A small, type-first CLI builder for Node that wraps the platform's own argument parser instead of reimplementing one.

[GitHub repo](https://github.com/unjs/citty) ·
[npm](https://www.npmjs.com/package/citty) ·
[License: MIT](https://github.com/unjs/citty/blob/main/LICENSE)

## Overview

citty is a command-line-interface builder from the UnJS collective — the same
group behind Nuxt's tooling, unbuild, ofetch, and h3[^1]. Its pitch is narrow
and honest: define commands and their arguments as plain objects, get typed
`args` back, and let the library handle usage rendering, sub-command routing,
and graceful error exit. It is the CLI layer used internally by `nuxi` (the
Nuxt CLI) and a number of other UnJS binaries, which is where most of its
production mileage comes from.

The defining design choice is that citty does not ship its own argument
tokenizer. Parsing is delegated to Node's built-in `util.parseArgs`[^2], and
citty layers typecasting, defaults, positional handling, aliases, and help
generation on top. This keeps the dependency count at zero and the bundle
small, but it also means citty inherits `parseArgs`'s limits — it is a
convenience and type-safety layer, not a parser in its own right.

The second thing to know is that citty is still pre-1.0. The latest published
version is 0.2.2 (April 2026)[^3], and the package sat on the 0.1.x line for
roughly two years before the 0.2.x series introduced the plugin API. It is
maintained and shipping, but the version number is an accurate signal that the
public API may still shift between minor releases; treat it as "stable enough
for the UnJS ecosystem, pin your version" rather than "frozen."

## Getting Started

```sh
npx nypm add citty     # or: npm i citty
```

```js
// index.mjs
import { defineCommand, runMain } from "citty";

const main = defineCommand({
  meta: { name: "hello", version: "1.0.0", description: "My CLI App" },
  args: {
    name: { type: "positional", description: "Your name", required: true },
    friendly: { type: "boolean", description: "Use friendly greeting", alias: ["f"] },
  },
  run({ args }) {
    console.log(`${args.friendly ? "Hi" : "Greetings"} ${args.name}!`);
  },
});

runMain(main);
```

```sh
node index.mjs john --friendly
# Hi john!
```

`args` is fully typed from the definition: `args.name` is a `string`,
`args.friendly` a `boolean`. `--help`/`-h` and `--version`/`-v` are wired up
automatically unless your own args collide with those names.

## Architecture / How It Works

A command is a plain object (`CommandDef`) passed to `defineCommand`, which is
a pure type helper — it returns its input unchanged and exists only to give the
TypeScript inference something to hang on. The runtime work happens in
`runCommand` / `runMain`:

1. **Parse** — raw argv is handed to `util.parseArgs` with an options map
   derived from your `args` definition. citty then applies typecasting (`enum`,
   boolean `--no-` negation, defaults) and separates positionals.
2. **Route** — if `subCommands` is present, the first positional selects a
   sub-command and parsing recurses into it. Sub-commands nest arbitrarily.
3. **Resolve** — `meta`, `args`, and `subCommands` are each `Resolvable<T>`:
   a value, a Promise, a function, or an async function. This is what enables
   lazy loading — a sub-command can be `() => import("./sub.mjs")` so only the
   command actually invoked is loaded.
4. **Run** — plugin `setup` hooks run (in order), then the command's own
   `setup`, then `run`, then `cleanup` and plugin `cleanup` (in reverse).
   `cleanup` is guaranteed even if `run` throws.

Usage text (`renderUsage`/`showUsage`) is generated from the same definition,
so help output stays in sync with the declared args by construction. The
`enum` type, `alias`, `valueHint`, and camelCase access to kebab-case flags
(`args["output-dir"]` === `args.outputDir`) are all citty conveniences layered
above `parseArgs`, which itself only understands strings and booleans.

The whole thing is one small ESM package with no runtime dependencies. That is
its main structural strength and the reason it composes cleanly inside larger
UnJS binaries.

## Production Notes

- **`util.parseArgs` is the floor and the ceiling.** citty needs a Node
  version where `parseArgs` is available and behaves as expected (Node 18.3+,
  practically Node 18/20+). Anything `parseArgs` cannot express — variadic
  options, complex `--opt=a,b` list semantics, mid-argv option/positional
  interleaving edge cases — is not something citty magically fixes. If you hit
  a parsing corner case, check `parseArgs`'s behavior first.
- **Pre-1.0, so pin it.** The 0.1 → 0.2 jump added the plugin API and moved
  some internals. There is no semver-major stability promise yet; pin an exact
  version in tools you publish and read the changelog before bumping.
- **Typed args are a compile-time convenience, not runtime validation.** citty
  casts types and applies defaults but does not do rich validation (ranges,
  formats, mutually-exclusive groups). For anything beyond type/required/enum,
  validate inside `run` or reach for a schema library.
- **Lazy sub-commands are the intended scaling path.** For a CLI with many
  commands, define each `subCommands` entry as a dynamic `import()` so startup
  does not pay for loading every command's module. This is the pattern `nuxi`
  itself uses and is the main reason to prefer citty over a flat parser for
  large tools.
- **Small surface, small ecosystem.** There is no plugin marketplace, no
  built-in prompts/spinners/tables. citty deliberately does only argument
  definition, routing, and usage; pair it with UnJS siblings (`consola` for
  logging, `unbuild` for bundling) or standalone libraries for the rest.

## When to Use / When Not

**Use when:**
- You want typed `args` from a declarative definition without a heavy framework.
- You are building a CLI in or near the UnJS/Nuxt ecosystem and want consistent, zero-dependency tooling.
- You have nested sub-commands and want lazy loading to keep startup fast.
- You value a tiny bundle and are comfortable tracking a pre-1.0 dependency.

**Avoid when:**
- You need battle-tested API stability and a large plugin ecosystem today — reach for a mature 1.x-plus tool.
- Your CLI needs rich interactive prompts, tables, or spinners out of the box — citty does not provide them.
- You need parsing features beyond what `util.parseArgs` supports and don't want to work around its constraints.
- You must run on old Node versions without a usable `parseArgs`.

## Alternatives

- yargs/yargs — the mature, feature-heavy standard; use it when you need every parsing feature and a proven 1.x API and don't mind the size.
- tj/commander.js — the most widely used minimal-ish CLI framework; use it when you want a stable, ubiquitous option with a large community.
- oclif/oclif — Salesforce's plugin-based framework; use it when you're building a large, extensible CLI product with a plugin ecosystem and generators.
- 75lb/command-line-args — a thin, unopinionated arg parser; use it when you want parsing only and will build routing and help yourself.
- google/zx or antfu's cac — use cac when you want a similarly small, fast builder with a slightly different (chainable) API and its own maturity track.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2023-03-14 | First real publish under unjs/citty[^3]. |
| 0.1.0 | 2023-03-31 | Early stable-ish API; defineCommand/runMain shape settled. |
| 0.1.6 | 2024-02-14 | Last of the long-lived 0.1.x line. |
| 0.2.0 | 2026-01-20 | Plugin API (`defineCittyPlugin`) and internal reworks[^3]. |
| 0.2.2 | 2026-04-01 | Current release as of this writing[^3]. |

## References

[^1]: UnJS — the ecosystem citty belongs to. https://unjs.io/
[^2]: Node.js docs, `util.parseArgs`. https://nodejs.org/api/util.html#utilparseargsconfig
[^3]: citty on npm (version/time metadata). https://www.npmjs.com/package/citty

## Tags

typescript, cli, cli-builder, argument-parser, nodejs, unjs, zero-dependency, esm, developer-tools, command-line
