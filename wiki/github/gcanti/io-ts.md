# gcanti/io-ts

> Runtime type system for TypeScript where a codec is a value: decode untrusted IO into statically-typed data and get the type back for free.

[GitHub repo](https://github.com/gcanti/io-ts) ·
[Documentation](https://gcanti.github.io/io-ts/) ·
[License: MIT](https://github.com/gcanti/io-ts/blob/master/LICENSE)

## Overview

io-ts is a runtime validation library by Giulio Canti, first released in 2017[^1]. Its premise is that a TypeScript type erased at compile time is useless for data crossing an IO boundary — HTTP responses, `JSON.parse`, form input, env vars — so you should build a *codec* (a runtime value of type `t.Type<A, O, I>`) that can both validate input and hand you a compile-time type via `t.TypeOf<typeof codec>`. You write the schema once; the static type is derived, never duplicated.

The defining characteristic — and the reason its popularity peaked and then receded — is that io-ts is built on `fp-ts`[^2], Canti's functional-programming standard library, and is a required peer dependency. `decode` does not throw and does not return a boolean; it returns an `Either<Errors, A>`, and idiomatic usage means folding that `Either`, composing with `pipe`, and understanding the `fp-ts` vocabulary. This buys principled, composable error handling but imposes a real learning curve on teams that do not already speak `fp-ts`.

io-ts arrived years before "TypeScript-first schema validation" became a crowded category. It proved the pattern; it did not win the ergonomics contest. Since around 2023 Canti's attention shifted to the Effect ecosystem, whose `@effect/schema` (now `effect/Schema`) is the direct spiritual successor, and the repository has settled into low-activity maintenance — the last commit to `master` was in December 2024[^3]. At ~6.8k stars it remains widely deployed in existing codebases, but new projects overwhelmingly reach for `zod` instead.

## Getting Started

```sh
npm i io-ts fp-ts
```

```ts
import * as t from "io-ts";
import { isRight } from "fp-ts/Either";

// A codec is a value.
const User = t.type({
  id: t.number,
  name: t.string,
  roles: t.array(t.string),
});

// Derive the static type — do NOT write an interface by hand.
type User = t.TypeOf<typeof User>; // { id: number; name: string; roles: string[] }

const result = User.decode(JSON.parse('{"id":1,"name":"Ada","roles":["admin"]}'));

if (isRight(result)) {
  result.right.name; // typed as string
} else {
  // result.left is an array of decode errors; PathReporter turns it into strings
}
```

```ts
import { PathReporter } from "io-ts/PathReporter";
console.log(PathReporter.report(User.decode({ id: "1" })));
// [ 'Invalid value "1" supplied to : { id: number, ... }/id: number', ... ]
```

## Architecture / How It Works

The core abstraction is `Type<A, O = A, I = unknown>`, carrying three functions: `is` (a type guard `I => boolean`), `decode` (`I => Either<Errors, A>`), and `encode` (`A => O`). `A` is the runtime-validated type, `O` the serialized/output type, `I` the input type. Because encode exists, codecs are bidirectional — `t.type` produces an identity encoder, but branded and transforming codecs (e.g. `DateFromISOString` in `io-ts-types`[^4]) use the `O` slot to round-trip.

Codecs compose through combinators rather than a fluent builder: `t.type` (required props), `t.partial` (optional props), `t.intersection` (to combine the two), `t.array`, `t.record`, `t.union`, `t.literal`, `t.keyof`, `t.tuple`, `t.brand` (nominal types), and `t.recursion` for self-referential schemas. There is no chaining API; you nest and intersect. `t.exact` is required to strip excess properties — by default codecs are non-exact and pass unknown keys through.

Errors are structured values, not strings. `decode` returns `Either<t.Errors, A>` where `t.Errors` is a non-empty array of `ValidationError`, each carrying the full `Context` (the path of codecs traversed) and the offending value. Reporters (`PathReporter`, or a custom fold over the context) turn that tree into human output. This is more faithful than a flat message list but means you cannot read an error without importing a reporter or writing one.

The library ships two generations side by side. The **stable** surface is the original `index.ts` module described above. The **experimental** modules introduced in 2.2 — `Decoder`, `Encoder`, `Codec`, `Eq`, `Schema` — split decoding from encoding and were designed to address inference and performance limits of the original design. The README states plainly that these are *independent and backward-incompatible* with the stable module and may change without notice[^5]. That unresolved split — a stable API the author had moved past, and an experimental one never promoted to stable — is a structural reason the project stalled rather than converging.

## Production Notes

- **fp-ts is not optional.** Even minimal use pulls `fp-ts` into your dependency tree and your mental model. Handling a `decode` result correctly means working with `Either` (`isRight`/`fold`/`pipe`). Teams that adopted io-ts "just for validation" without buying into `fp-ts` consistently found the ergonomics heavier than expected.
- **Non-exact by default is a footgun.** `t.type` lets unknown properties pass through untouched. If you rely on validation to reject unexpected fields (security-sensitive input), you must wrap in `t.exact` or `t.strict`; forgetting this is a common source of silent over-permissive decoding.
- **Optional vs. required is a hard split.** There is no per-field optional marker; optional properties live in a separate `t.partial` codec that you `t.intersection` with the required `t.type`. Deep schemas become verbose, and the intersection pattern is the single most-asked io-ts question.
- **Union decode errors are noisy.** For large `t.union`s (e.g. discriminated unions), a failed decode reports failures against *every* member unless you reach for `t.union` with a well-chosen tag, producing long, hard-to-read error dumps.
- **Recursive types need `t.recursion` plus a manual type annotation** — TypeScript cannot infer the recursive `TypeOf`, so you annotate the type yourself and keep it in sync, partially defeating the derive-the-type promise.
- **Maintenance status.** With the last commit in December 2024 and 161 open issues[^3], treat io-ts as feature-frozen. It works and is not abandoned in the "deleted" sense, but do not expect new features, and evaluate the migration path before starting a greenfield project on it.

## When to Use / When Not

**Use when:**
- Your team already uses `fp-ts` (or Effect) and wants validation that speaks the same `Either`/`pipe` idiom.
- You need true bidirectional codecs (decode *and* encode/serialize) with custom transformations, not just input validation.
- You value structured, path-aware error values over convenience strings.
- You are maintaining an existing io-ts codebase and stability matters more than new features.

**Avoid when:**
- You want the shortest path from "TypeScript type" to "runtime validator" without a functional-programming prerequisite — `zod` is the near-universal default here.
- You are starting new and want an actively evolving library; io-ts is effectively frozen and its author has moved to Effect Schema.
- Bundle size is critical and you cannot afford to ship `fp-ts` alongside.

## Alternatives

- colinhacks/zod — the de facto default now; use instead when you want a chainable, fp-ts-free API and the largest ecosystem of adapters.
- Effect-TS/effect — use instead when you want io-ts's designed successor (`effect/Schema`): bidirectional codecs by the same author, integrated with the Effect runtime.
- sinclairzx81/typebox — use instead when you need JSON Schema output for OpenAPI/AJV rather than TypeScript-first inference.
- pelotom/runtypes — use instead when you want combinator-style validation with a lighter footprint and no fp-ts dependency.
- ajv-validator/ajv — use instead when your schema source of truth is JSON Schema, not TypeScript types.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017 | Initial release; codec-as-value core[^1]. |
| 1.0 | 2018 | First stable line on the `Type<A, O, I>` model. |
| 2.0 | 2019 | Major version; API cleanup, dropped some v1 combinators. |
| 2.2 | 2020 | Experimental `Decoder`/`Encoder`/`Codec`/`Schema` modules, backward-incompatible with stable[^5]. |
| 2.2.x | 2024-12 | Last commit to `master`; project enters low-activity maintenance[^3]. |

## References

[^1]: gcanti/io-ts repository and history. https://github.com/gcanti/io-ts
[^2]: fp-ts — required peer dependency and functional-programming foundation. https://github.com/gcanti/fp-ts
[^3]: Repository metadata via GitHub API: last push 2024-12-10, 161 open issues, MIT license, ~6.8k stars (fetched 2026-07). https://github.com/gcanti/io-ts
[^4]: io-ts-types — additional codecs such as `DateFromISOString`. https://github.com/gcanti/io-ts-types
[^5]: io-ts README, "Experimental modules (version 2.2+)": modules are independent and backward-incompatible with the stable API. https://github.com/gcanti/io-ts/blob/master/README.md
[^6]: Effect Schema — the spiritual successor by the same author. https://effect.website/docs/schema/introduction/

## Tags

typescript, validation, runtime-types, type-inference, fp-ts, functional-programming, codec, decoder, schema, io-boundary
