# colinhacks/zod

> TypeScript-first schema validation where one schema declaration yields both a runtime validator and a static type.

[GitHub repo](https://github.com/colinhacks/zod) ·
[Official website](https://zod.dev) ·
[License: MIT](https://github.com/colinhacks/zod/blob/main/LICENSE)

## Overview

Zod is a schema-declaration and validation library for TypeScript, authored by Colin McDonnell and first released in 2020[^1]. Its defining idea: you write a schema once, and Zod gives you both a runtime parser (which throws or returns a result on invalid data) and a statically inferred TypeScript type via `z.infer<>`. This single-source-of-truth model eliminated a category of drift where hand-written interfaces and hand-written validators disagreed. It became, for practical purposes, the default validation layer of the TypeScript ecosystem — tRPC, React Hook Form resolvers, T3 stack, and a large share of form/env/API-boundary code assume it.

The library's central tension is that it does its type inference entirely in the TypeScript type system. Schemas are ordinary values whose types are computed at compile time, which is why inference "just works" — and also why deeply nested schemas, large discriminated unions, and long `.extend()`/`.merge()` chains can inflate `tsc` type-instantiation counts and slow editor responsiveness in big codebases. Zod 4 (2025) was a substantial internal rewrite aimed squarely at this: fewer type instantiations, faster parse throughput, a smaller core, and a tree-shakable `zod/mini` variant[^2].

Zod is also a co-author of Standard Schema[^3], a shared `~standard` interface that lets libraries (tRPC, form resolvers, etc.) accept any conforming validator — Zod, Valibot, or ArkType — without a per-library adapter.

## Getting Started

```sh
npm install zod
```

```ts
import * as z from "zod";

const User = z.object({
  username: z.string().min(3),
  xp: z.number().int().nonnegative(),
  email: z.email().optional(),
});

// throws ZodError on invalid input
const user = User.parse({ username: "billie", xp: 100 });

// non-throwing variant — discriminated union result
const result = User.safeParse({ username: "x", xp: -1 });
if (!result.success) {
  console.error(result.error.issues);
} else {
  result.data.username; // fully typed
}

// the schema IS the type
type User = z.infer<typeof User>; // { username: string; xp: number; email?: string }
```

Async schemas (async refinements/transforms) require `.parseAsync()` / `.safeParseAsync()`.

## Architecture / How It Works

Every schema is an instance of a `ZodType` subclass (`ZodString`, `ZodObject`, `ZodUnion`, …). The runtime behavior and the static type live on the same object: the value carries a parse method, and the class's generic parameters carry the inferred input/output types that `z.infer` reads.

- **Parsing.** `.parse()` walks the schema tree against the input, collecting `ZodIssue` entries. On failure it aggregates them into a single `ZodError` rather than failing fast, so a form can surface every field error at once. `.safeParse()` wraps the same pass in a `{ success, data | error }` result to avoid `try/catch`.
- **Input vs output types.** Because `.transform()` and coercion can change a value's shape, a schema has a distinct input type (`z.input`) and output type (`z.output` = `z.infer`). This split matters wherever a schema both validates and reshapes data.
- **Immutability.** Methods like `.optional()`, `.min()`, `.extend()` return a new schema instance rather than mutating — chaining composes without aliasing surprises.
- **Refinements and transforms.** `.refine()` adds custom predicate checks; `.transform()` maps the parsed value. Either can be async, which is what forces the `*Async` parse variants.
- **Zod 4 internals.** Version 4 re-architected the type-level machinery to cut instantiations, made the core bundle small (~2 kB gzipped for the string/object path), added first-class `z.toJSONSchema()` conversion, and shipped `zod/mini` — a function-style, tree-shakable API (`z.optional(z.string())` instead of `z.string().optional()`) for bundle-size-sensitive targets[^2].

The coupling worth understanding: inference quality and compile cost are two sides of the same mechanism. You cannot get Zod's ergonomics without putting real work on `tsc`.

## Production Notes

- **TypeScript compile cost is the recurring footgun.** Large schema graphs — many `.extend()`/`.merge()`, wide `z.discriminatedUnion()`, recursive schemas — can dominate `tsc` time and degrade IDE latency. Zod 4 materially improved this, but on Zod 3 the standard mitigations still apply: split large schemas into named intermediates, prefer `.pick()`/`.omit()` over deep re-composition, and watch `tsc --extendedDiagnostics` instantiation counts.
- **Runtime is not free.** Zod's per-parse work is meaningful on hot paths (high-RPS request validation, tight loops). Compile-to-validator libraries (TypeBox + AJV) and lighter parsers are faster at runtime; benchmark before validating in the hot path rather than at the boundary.
- **`z.coerce` is blunt.** `z.coerce.number()` runs `Number(input)`, so `""` becomes `0` and `"abc"` becomes `NaN` (then fails). For query strings and form data, be explicit about empty/absent handling instead of relying on coercion defaults.
- **Error shape changed in v4.** The issue object moved from `.error.errors` toward `.error.issues`, and formatting helpers were reworked (`z.treeifyError`, `.flatten()` semantics). Code that string-matched or reached into internal issue fields is the most fragile across the upgrade.
- **`.default()` / `.catch()` affect the input type.** A field with `.default()` becomes optional on input but present on output — a frequent source of "why is this optional?" confusion at API boundaries.
- **Zod 3 → 4 migration.** v4 ships in the same `zod` package; the import style shifted to `import * as z from "zod"`, and `zod/mini` is a separate entry point. Most schemas port unchanged, but custom error maps, error-object access, and any use of removed/renamed internals need review. Pin the version and read the migration guide before bumping a large codebase[^2].

## When to Use / When Not

**Use when:**
- You want one declaration to produce both runtime validation and a static type.
- You're validating trust boundaries — API request/response, env vars, form input, `localStorage`, third-party JSON.
- You're on tRPC, React Hook Form, or another tool that already speaks Zod (or Standard Schema).

**Avoid when:**
- You need maximum runtime throughput on a hot path — a compiled validator (TypeBox + AJV) or a lighter parser wins.
- Bundle size is critical and you can't adopt `zod/mini` — Valibot is smaller by design.
- You want JSON Schema as the source of truth and TS types as derived — that inverts Zod's model; reach for TypeBox instead.

## Alternatives

- fabian-hiller/valibot — modular, function-first API with aggressive tree-shaking; much smaller bundles when you use few features. Use instead when client bundle size dominates.
- sinclairzx81/typebox — schemas that are literally JSON Schema, validated via AJV. Use instead when you want fastest runtime validation or JSON Schema as the canonical artifact.
- arktypeio/arktype — validators written in near-TypeScript syntax with very fast inference and runtime. Use instead when you want the type-syntax ergonomics and top-tier performance.
- jquense/yup — older object-schema validator, less TypeScript-native inference. Use instead only in legacy codebases already built around it.
- gcanti/io-ts — codec-based, fp-ts-flavored. Use instead when your codebase is already committed to functional-programming idioms.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020 | Initial release — schema + inferred type model[^1]. |
| 2.0 | 2020 | API iteration on the early design. |
| 3.0 | 2021 | Long-lived stable line; ecosystem standard (tRPC, RHF, T3). |
| 3.22+ | 2023–2024 | Incremental additions; growing awareness of `tsc` cost at scale. |
| 4.0 | 2025 | Internal rewrite: fewer type instantiations, faster parsing, `zod/mini`, built-in JSON Schema output[^2]. |

## References

[^1]: Zod repository and documentation, Colin McDonnell (@colinhacks). https://github.com/colinhacks/zod
[^2]: "Introducing Zod 4" / migration guide, zod.dev. https://zod.dev/v4
[^3]: Standard Schema — shared validator interface (Zod, Valibot, ArkType). https://github.com/standard-schema/standard-schema

## Tags

typescript, javascript, schema-validation, runtime-validation, type-inference, static-types, parsing, form-validation, npm, library
