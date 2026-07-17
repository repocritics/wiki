# mattpocock/ts-reset

> A "CSS reset" for TypeScript — a set of declaration merges that fix long-standing weak spots in the built-in typings of common JavaScript APIs.

[GitHub repo](https://github.com/mattpocock/ts-reset) ·
[Official website](https://www.totaltypescript.com/ts-reset) ·
[License: MIT](https://github.com/mattpocock/ts-reset/blob/main/LICENSE.md)

## Overview

`ts-reset` is a single-import, runtime-free package that overrides parts of TypeScript's
standard library type definitions to make them stricter and more ergonomic. It was written by
Matt Pocock (Total TypeScript) and published in February 2023[^1], where it circulated widely
because it packaged fixes the community had been hand-rolling for years. The name is deliberate:
like a browser CSS reset, it smooths over inconsistent defaults so every project starts from a
saner baseline.

The concrete problems it targets are well known. `JSON.parse` and `Response.json()` return `any`,
which silently disables type-checking on any value derived from parsed data. `.filter(Boolean)`
does not remove `null`/`undefined` from the resulting element type. `Array.prototype.includes`
and `Set.prototype.has` reject arguments that aren't already members of the (often narrowed)
element type, which is the opposite of how membership tests are used in practice. `ts-reset`
replaces `any` with `unknown` in the first two cases and widens the argument types in the latter.

The defining tension is that these are opinionated changes to global types, and one of them
trades away type safety on purpose. Turning `.includes`/`.has` arguments into a wider type means
TypeScript will no longer catch "you're checking for a value that can't be in this array." Most
users consider that a good trade; some do not. Because the changes are global augmentations, the
package is meant for application codebases, not libraries — see Production Notes.

## Getting Started

```bash
npm install --save-dev @total-typescript/ts-reset
```

Create a single `.d.ts` file that is part of your compilation (e.g. `reset.d.ts` at the project
root) and import the package once:

```ts
// reset.d.ts — do this in exactly ONE file, project-wide
import "@total-typescript/ts-reset";
```

After that, the improved types apply everywhere:

```ts
// Before ts-reset: data is `any`. After: data is `unknown`.
const data = JSON.parse(rawString);
//    ^? unknown  — you must narrow it before use

const values = [1, null, 2, undefined].filter(Boolean);
//    ^? number[]  — null/undefined removed, not number|null|undefined[]

const roles = ["admin", "user"] as const;
if (roles.includes(inputRole)) { /* inputRole: string is accepted */ }
```

You can also opt in to individual rules instead of the recommended bundle:

```ts
import "@total-typescript/ts-reset/filter-boolean";
import "@total-typescript/ts-reset/json-parse";
import "@total-typescript/ts-reset/fetch";
import "@total-typescript/ts-reset/array-includes";
import "@total-typescript/ts-reset/set-has";
import "@total-typescript/ts-reset/is-array";
```

## Architecture / How It Works

`ts-reset` ships **no runtime code**. Every file in the package is a `.d.ts` declaration file that
uses TypeScript's interface merging and global augmentation to re-declare methods on built-in
interfaces like `Body`, `JSON`, `Array`, `ReadonlyArray`, and `Set`. When you import it, those
declarations merge into the global scope and take precedence over the ones in `lib.es*.d.ts`.

Because the mechanism is pure declaration merging, there is zero bundle-size cost, zero runtime
behavior change, and nothing to tree-shake — the package disappears entirely at compile time. What
changes is only what the type-checker infers. This also means it can only *tighten or widen* method
signatures TypeScript already ships; it cannot add new lint-style diagnostics or change emitted JS.

The default `import "@total-typescript/ts-reset"` applies a curated "recommended" set of rules, not
every rule the package contains. Each rule lives at its own subpath (`/json-parse`, `/fetch`,
`/filter-boolean`, `/array-includes`, `/set-has`, `/is-array`, and others) so you can compose your
own subset. The rules are independent of one another; importing one does not pull in the rest.

## Production Notes

- **Import exactly once, in a `.d.ts` that is included in the build.** If tsconfig `include`
  patterns exclude your reset file, nothing happens and there is no error. Importing it in multiple
  files is harmless but pointless. Putting it inside a regular `.ts` module is fine as long as the
  side-effect import isn't dropped.
- **Do not use it in a published library.** Global augmentations leak to every downstream consumer
  of your package. A library that imports `ts-reset` silently rewrites `JSON.parse`, `.includes`,
  and friends for anyone who installs it — a surprising and often unwanted side effect. The docs are
  explicit that this is an application-level tool[^2].
- **`unknown` is a migration cost, not a free win.** Switching `JSON.parse`/`fetch().json()` from
  `any` to `unknown` is the most valuable rule and the most disruptive: every call site that fed
  parsed data into typed code will now error until you add narrowing (a cast, a type guard, or a
  validator such as Zod). On a large existing codebase, adopting this rule is a real refactor.
- **The `.includes`/`.has` widening deliberately reduces safety.** It stops TypeScript from
  flagging membership checks against values outside the element type. That is the intended behavior,
  but if your team relies on that check as a correctness signal, opt out of `array-includes`/`set-has`
  and keep the rest.
- **Requires a reasonably modern TypeScript** (the docs list a 4.x-era minimum). It does not pin or
  patch your TS version; it only augments whatever `lib` your tsconfig already loads.
- **Still pre-1.0.** The package sits in the `0.x` range and evolves slowly; it is stable in
  practice (small surface, no runtime) but has not committed to a 1.0 API. Pin the version if you
  care about a specific rule set.

## When to Use / When Not

**Use when:**
- You maintain an application (not a library) and want stricter, more ergonomic std-lib types.
- You are already validating external data and want `unknown` to force that discipline at the
  boundary.
- You want `.filter(Boolean)` and `Array.isArray` to narrow correctly without hand-written helpers.

**Avoid when:**
- You ship a library or shared package — global augmentation would infect consumers.
- Your team depends on `.includes`/`.has` rejecting non-member arguments as a type-safety check.
- You cannot afford the one-time refactor that `any` → `unknown` on `JSON.parse`/`fetch` triggers.

## Alternatives

- colinhacks/zod — runtime schema validation; the natural companion for turning ts-reset's
  `unknown` returns into typed data. Use instead of ts-reset when your real need is validating
  external input, not retyping the std lib.
- sindresorhus/type-fest — a large collection of additive utility types. Use when you want opt-in
  helper types rather than global overrides of existing methods.
- millsp/ts-toolbelt — type-level programming utilities. Use for advanced type manipulation, not
  for fixing built-in method signatures.
- Hand-written `.d.ts` augmentations — the zero-dependency path; re-declare the handful of methods
  you care about yourself. Use when you want full control and no third-party global changes.
- `@tsconfig/strictest` — stricter compiler flags. Use alongside (not instead of) ts-reset; it
  tightens config, ts-reset tightens the std-lib types.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-02 | First public release by Matt Pocock; framed as a "TypeScript CSS reset"[^1]. |
| 0.x | 2023–2026 | Iterative rule additions and granular subpath imports; remains pre-1.0. |
| — | 2026-04 | Most recent commit to `main` at time of writing. |

## References

[^1]: Matt Pocock, "ts-reset" announcement and docs — Total TypeScript. https://www.totaltypescript.com/ts-reset
[^2]: `ts-reset` README and documentation (application-only guidance, rule list). https://github.com/mattpocock/ts-reset

## Tags

typescript, type-safety, dts, declaration-merging, developer-tooling, javascript, strict-typing, json-parse, fetch, library
