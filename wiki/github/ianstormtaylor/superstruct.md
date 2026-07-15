# ianstormtaylor/superstruct

> A composable, function-first runtime validation library for JavaScript and TypeScript, built around small tree-shakeable primitives rather than a fluent builder.

[GitHub repo](https://github.com/ianstormtaylor/superstruct) ·
[Official website](https://docs.superstructjs.org) ·
[License: MIT](https://github.com/ianstormtaylor/superstruct/blob/main/License.md)

## Overview

Superstruct validates unknown data at runtime against declared "structs" and, in
TypeScript, narrows the value's static type as a side effect. It was written by
Ian Storm Taylor (of Slate and Segment) and first published in 2017[^1], predating
most of the current generation of TypeScript-first validators. Its stated design
goal is a small, unopinionated core: it ships validators for native JavaScript
types and expects you to compose everything else yourself, in one place, from
plain functions.

The defining stylistic choice — and the source of its main tradeoff — is that
structs are built by *calling and nesting functions* (`object({ id: number() })`)
rather than chaining methods off a builder (`z.object({...}).min(1)`). This keeps
each primitive independently importable and the bundle small, but it makes some
patterns (refinements, coercion, defaults) read as wrapper functions applied
around a struct instead of fluent calls, which many developers find less
discoverable. That ergonomic gap is the practical reason Zod overtook Superstruct
in mindshare after 2020, even though the two solve the same problem.

The library reached a stable 1.0 line[^2] and, more recently, restructured its
distribution around a scoped `@superstruct/core` package published to JSR while
keeping the classic `superstruct` npm package as the entry point[^3].

## Getting Started

```bash
npm install superstruct
```

```ts
import { assert, object, number, string, array, is, Infer } from 'superstruct'

const Article = object({
  id: number(),
  title: string(),
  tags: array(string()),
  author: object({ id: number() }),
})

type Article = Infer<typeof Article>   // static type derived from the struct

const data: unknown = JSON.parse(input)

assert(data, Article)   // throws a StructError with a path on the first failure
// after assert(), TypeScript treats `data` as Article

if (is(data, Article)) {
  // boolean form; narrows `data` inside the block without throwing
}
```

Use `validate(data, struct)` to get `[error, value]` instead of throwing, and
`create(data, struct)` to run coercions (defaults, type conversion) as part of
validation.

## Architecture / How It Works

A `Struct` is an object holding a `validator` function plus optional `coercer`,
`refiner`, and a `schema` describing nested structs. Every builder — `string()`,
`object()`, `array()`, `union()`, `record()`, `tuple()` — returns one of these.
Validation walks the value and the struct together, yielding a lazy iterator of
failures; each failure carries a `path` (array of keys) and the offending
`value`, which is what makes Superstruct's errors more actionable than the
boolean/string errors of older validators.

Three orthogonal concerns are layered as separate wrapper functions rather than
methods:

- **Refinement** — `refine(struct, name, predicate)` adds an extra constraint
  (e.g. positive number, non-empty string) on top of an existing struct.
- **Coercion** — `coerce(struct, condition, handler)` and helpers like
  `defaulted()` and `trimmed()` transform input *before* validation; they only
  run under `create()`, never under `assert()`/`is()`.
- **Custom types** — `define(name, validator)` builds a leaf struct from a
  boolean predicate. Because the predicate returns only `true`/`false`, custom
  types built this way produce a generic failure message unless you return a
  string reason.

TypeScript inference is derived structurally from the schema via the `Infer<>`
helper; there is no code generation step. Unions validate by trying each branch,
which means a failed `union()` reports the failures of *every* branch, and
error output for large unions can be hard to read. The library is dependency-free
and ships ESM, CommonJS, and UMD builds.

## Production Notes

**Maintenance has gone quiet.** The default branch's last push was in late
2024[^4], and there is a backlog of open issues. The core is stable and widely
depended upon, so "unmaintained" overstates it — but treat Superstruct as
feature-complete rather than actively evolving, and do not expect fast turnaround
on new feature requests or ecosystem-tracking changes.

**Coercion is opt-in and easy to miss.** `assert()` and `is()` do *not* apply
coercers or defaults; only `create()` does. Teams that validate with `assert()`
and expect `defaulted()` values to appear will silently get undefined fields.

**Union error messages are noisy.** Discriminated data is better modeled with a
narrowing check before validation, or by keeping unions small, because a union
failure surfaces the failures of all branches at once.

**Bundle size is the standing advantage.** The function-per-primitive design is
genuinely tree-shakeable, so apps that import a handful of validators pull in far
less code than method-builder libraries. If minimized client-side validation
weight matters, this is the reason to still reach for Superstruct.

**Distribution moved.** Recent versions publish `@superstruct/core` on JSR while
`superstruct` remains the npm package[^3]. Pin versions and read release notes
before upgrading across the 1.x line, as the packaging changed even though the
public API stayed largely stable.

## When to Use / When Not

**Use when:**
- You want runtime validation with detailed, path-carrying errors and a small
  bundle footprint.
- You prefer composing validators from plain, individually-importable functions.
- You need custom, application-specific types defined once and reused.

**Avoid when:**
- You want the largest ecosystem, most examples, and most third-party
  integrations — that is Zod today.
- You want a fluent, method-chained API with rich inline refinements.
- You need an actively, rapidly maintained project tracking the latest TS
  features.

## Alternatives

- colinhacks/zod — the de facto default; fluent chained API, huge ecosystem. Use instead when you want mindshare, integrations, and inline refinements over minimal bundle size.
- fabian-hiller/valibot — modular function-first design like Superstruct but smaller and actively maintained. Use instead when you like the functional style but want a current, size-optimized library.
- jquense/yup — object-schema validation popular in form libraries. Use instead when you're wiring validation into Formik/React forms.
- gcanti/io-ts — codec-based validation with an fp-ts foundation. Use instead when you want encode/decode round-tripping and a functional-programming stack.
- ajv-validator/ajv — JSON Schema compiler. Use instead when your contract is JSON Schema (OpenAPI, config validation) and you need maximum throughput.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2017-11 | First published; function-composition validation API[^1]. |
| 0.x | 2018–2022 | Long 0.x line; rewritten in TypeScript, `Infer<>` inference added. |
| 1.0 | 2023 | Stable 1.0 API line[^2]. |
| 1.x | 2024 | `@superstruct/core` published to JSR; last active pushes[^3][^4]. |

## References

[^1]: superstruct repository, created 2017-11-23. https://github.com/ianstormtaylor/superstruct
[^2]: superstruct releases (1.0 stable line). https://github.com/ianstormtaylor/superstruct/releases
[^3]: `@superstruct/core` on JSR (scoped distribution). https://jsr.io/@superstruct/core
[^4]: Repository default branch `main`, last push 2024-10-01 (GitHub API, retrieved 2026-07). https://github.com/ianstormtaylor/superstruct

## Tags

typescript, javascript, validation, schema, runtime-validation, type-inference, data-validation, structs, composable, library
