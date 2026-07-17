# sindresorhus/type-fest

> A public-domain collection of compile-time TypeScript utility types — the de facto standard library for type-level programming.

[GitHub repo](https://github.com/sindresorhus/type-fest) ·
npm: `type-fest` ·
[License: CC0-1.0](https://github.com/sindresorhus/type-fest/blob/main/license)

## Overview

type-fest is a dependency-free package of TypeScript types that Sindre Sorhus began publishing in 2019[^1]. It ships no runtime code — the package is a set of `.d.ts` declaration files and nothing else. You install it to get types like `Except`, `Merge`, `PartialDeep`, `ReadonlyDeep`, `RequireAtLeastOne`, `Paths`, and `Get` that TypeScript's own standard library (`Partial`, `Pick`, `Omit`, `Record`) does not provide. The stated premise of the README is that "many of the types here should have been built-in"[^2].

The license is CC0-1.0 — a public-domain dedication rather than a typical software license. This is deliberate: the author invites users to copy-paste individual type definitions directly into their own code with no attribution, treating the package as a menu as much as a dependency[^2]. For a types-only library this is low-risk, since there is no runtime surface to keep in sync.

The defining tension is cost, not correctness. Many of the more useful types (`PartialDeep`, `ReadonlyDeep`, `Paths`, `Get`, `Schema`, `MergeDeep`) are recursive conditional/mapped types. They are correct, but they are also work the TypeScript compiler must do on every check, and the deep ones can hit the compiler's instantiation-depth ceiling on large inputs. type-fest gives you expressiveness that TypeScript's built-ins withhold partly for performance reasons — and you inherit that performance cost.

## Getting Started

```sh
npm install type-fest
```

Requires a recent TypeScript (the current line targets TypeScript >=5.9), ESM, and `"strict": true` in `tsconfig.json`[^2]. Import as `type` — the package has no value exports.

```ts
import type {Except, PartialDeep, Merge} from 'type-fest';

type User = {
	id: string;
	name: string;
	password: string;
};

// Remove a key at the type level (like Omit, but stricter about excess keys)
type PublicUser = Except<User, 'password'>;
//=> {id: string; name: string}

// Deeply optional — useful for PATCH-style update payloads
type UserPatch = PartialDeep<User>;

// Merge, with the second type's keys winning on conflict
type WithRole = Merge<User, {role: 'admin' | 'user'}>;
```

For the runtime counterparts of some of these types (type-guard functions, safe `Object.keys`, etc.), the author maintains a separate package, `ts-extras`[^3].

## Architecture / How It Works

There is no build output in the conventional sense. Each type lives in its own file under `source/` (for example `source/except.d.ts`, `source/partial-deep.d.ts`), and `index.d.ts` re-exports them. Installing the package adds declaration files to `node_modules`; nothing is emitted into your bundle and nothing runs.

The types are built almost entirely from TypeScript's type-level primitives: conditional types (`T extends U ? X : Y`), mapped types (`{[K in keyof T]: ...}`), `infer`, recursive template-literal types (used by `Paths`, `Get`, `Split`), and variadic tuple types (used by the array/tuple utilities). Categories in the README map to how the types are used: **Basic** (matchers like `Primitive`, `Class`, `TypedArray`), **Utilities** (the transforms — `Merge`, `SetOptional`, `PartialDeep`, `ConditionalKeys`, `Paths`), and **Type Guard** (`IsEqual`, `IsAny`, `IsNever`, `IsLiteral`, `If`) which return `true`/`false` type-level booleans used to compose other types.

The coupling that matters is to the TypeScript compiler version itself. Because these types push against the edges of what the type system can express, they routinely depend on features from recent TypeScript releases and on compiler behaviors that shift between versions. type-fest tracks the current TypeScript line closely and drops support for older compilers in major releases rather than maintaining backward compatibility[^4]. That is the price of living at the frontier of the type system.

## Production Notes

**Deep/recursive types are the footgun.** `PartialDeep`, `ReadonlyDeep`, `RequiredDeep`, `Paths`, `Get`, `Schema`, and `MergeDeep` recurse through nested object shapes. On large or highly-nested types they can produce `Type instantiation is excessively deep and possibly infinite` (TS error 2589), or simply slow `tsc` and the editor language server noticeably. When a build gets slow after adopting these, they are the first thing to profile — `tsc --generateTrace` will show it.

**It is a compile-time dependency only.** Zero bytes ship to production, zero runtime cost, no supply-chain execution surface. This also means it can live in `devDependencies` for library authors who don't re-export its types.

**Version churn is real.** type-fest bumps its minimum required TypeScript aggressively and does so in major versions; each major can raise the floor and occasionally change a type's inferred output as the compiler's inference improves. Upgrading type-fest sometimes requires upgrading TypeScript in the same step, and vice versa. Pin both and upgrade them together rather than letting a caret range drift.

**Excess-property strictness differs from built-ins.** Several types (`Except`, `Exact`, the `Require*` family) are intentionally stricter than the nearest built-in equivalent (`Omit`, plain object types). That strictness is the point, but it means a drop-in replacement of `Omit` with `Except` can surface new errors in previously-passing code.

**CC0 copy-paste is a supported path.** For a single type you need in one place, copying the definition out of `source/*.d.ts` is explicitly blessed and avoids adding a dependency at all[^2]. The tradeoff is you won't get upstream fixes.

## When to Use / When Not

**Use when:**
- You need type transforms TypeScript omits: deep partial/readonly, key-path access, conditional key selection, mutually-exclusive object shapes.
- You want a vetted, widely-depended-on source instead of hand-rolling fragile recursive conditional types.
- You want zero runtime weight — types-only, nothing in the bundle.

**Avoid when:**
- You only need `Partial`, `Pick`, `Omit`, `Record` — the built-ins already cover you; adding a dependency buys nothing.
- You need runtime validation, not just static types — a type says nothing at runtime; reach for a schema library instead.
- You must support an older, pinned TypeScript version — recent type-fest majors will refuse to type-check on it.
- Compile time is already a bottleneck and you'd be leaning on the deep recursive types heavily.

## Alternatives

- colinhacks/zod — use when you need runtime validation and want the static type inferred from a schema, not compile-only helpers.
- sindresorhus/ts-extras — use when you want the runtime functions (type guards, safe object helpers) that pair with these types.
- millsp/ts-toolbelt — use when you want a broader, more experimental type-level toolkit and accept heavier compiler cost.
- unional/type-plus — use when you want an alternative essential-types collection with a different API surface.
- microsoft/TypeScript — use the built-in utility types directly when basic transforms suffice and you want no dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-03-13 | First published by Sindre Sorhus[^1]. |
| 1.0 | 2021 | First stable major; consolidated the 0.x type set[^4]. |
| 2.0 | 2021 | Moved to pure ESM; raised minimum TypeScript[^4]. |
| 3.0 | 2022 | Further TypeScript floor bump and type additions[^4]. |
| 4.0 | 2023 | Major release; large batch of new types, higher TypeScript minimum[^4]. |

Version cadence is frequent — many minor releases add individual types between majors. Exact per-version dates and TypeScript floors are best read from the GitHub releases page[^4] rather than trusted from memory.

## References

[^1]: type-fest repository, created 2019-03-13 (GitHub API metadata). https://github.com/sindresorhus/type-fest
[^2]: type-fest README — install requirements, "should have been built-in", copy-paste / no-credit note. https://github.com/sindresorhus/type-fest#readme
[^3]: `ts-extras` — runtime companion package by the same author. https://github.com/sindresorhus/ts-extras
[^4]: type-fest releases and changelog (version history, TypeScript-version floors). https://github.com/sindresorhus/type-fest/releases

## Tags

typescript, types, type-level, utility-types, npm-package, dts, compile-time, developer-tools, sindresorhus, cc0
