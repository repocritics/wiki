# jquense/yup

> A schema builder for runtime value parsing, coercion, and validation — the pre-Zod default for JavaScript object validation, now living in Zod's shadow.

[GitHub repo](https://github.com/jquense/yup) ·
[License: MIT](https://github.com/jquense/yup/blob/master/LICENSE.md)

## Overview

Yup lets you declare a schema — `object({ name: string().required(), age: number().positive() })` — and then use it to parse (coerce values toward the declared type), assert (run tests over the value), or both. It has existed since 2014[^1], long predating the current wave of TypeScript-first validators, and for years it was the default choice for form and API validation in the React ecosystem, largely on the strength of its pairing with Formik (both authored by Jason Quense)[^2].

The defining tension in Yup is that it was designed as a runtime coercion library first and a TypeScript library second. The v1.0 release (2023) added static type inference via `InferType`[^3], but the inference is bolted onto an API that was never shaped around it: method *order* affects the inferred type, `.required()`/`.defined()`/`.nullable()` interact in non-obvious ways, and `mixed()`/`.when()`/`.test()` frequently need manual type annotations. Zod, which was built inference-first, has taken most of Yup's mindshare in new TypeScript projects. Yup remains widely used — tens of millions of npm downloads per week, still the validator Formik documents — but it is now a maintenance-mode incumbent rather than the growth story.

Yup's genuine strengths are its coercion model (it will happily turn `"24"` into `24` and `"2014-09-23T19:25:25Z"` into a `Date`), its async-native validation, and its expressiveness for interdependent field rules via `ref()` and `.when()`. If you need parsing/transformation as much as assertion, that model is a real fit.

## Getting Started

```bash
npm install yup
```

```ts
import { object, string, number, date, InferType } from 'yup';

const userSchema = object({
  name: string().required(),
  age: number().required().positive().integer(),
  email: string().email(),
  website: string().url().nullable(),
  createdOn: date().default(() => new Date()),
});

// validate() is async and returns the parsed (coerced) value
const user = await userSchema.validate({ name: 'jimmy', age: '24' });
// -> { name: 'jimmy', age: 24, createdOn: Date }  (age coerced from string)

type User = InferType<typeof userSchema>;
```

`cast()` coerces without asserting; `validateSync()` runs validation synchronously; `strict: true` skips coercion and validates the raw input as-is.

## Architecture / How It Works

A Yup schema is an immutable builder. Every method (`.required()`, `.min()`, `.transform()`) returns a *new* schema instance, so schemas can be shared and extended without mutation. Internally each schema holds two ordered lists: **transforms** (parsing steps that reshape the value) and **tests** (assertions that pass/fail without mutating).

Validation runs in two phases. First the transform pipeline coerces the input — an earlier transform's output feeds the next. Then the tests run over the coerced value. This ordering is why the README warns that inside a `transform` the value "is not guaranteed to be a valid type" (a number transform may receive `NaN`), whereas inside a `test` the value is already the correct type. Understanding this two-phase model is the key to using Yup correctly; most surprises come from expecting a test to see the raw input or a transform to see a clean type.

Cross-field logic is expressed with `ref()` (a lazy reference to a sibling or context value) and `.when()` (conditional schema selection based on other fields). Yup topologically sorts object field dependencies so refs resolve in the right order; `object.shape()` accepts a `noSortEdges` argument to break cycles manually. `lazy()` defers schema construction until validation time for recursive or value-dependent shapes.

TypeScript inference is layered on top via a set of generic flags (`TType`, `TFlags`, default/nullability markers) threaded through the schema types. `InferType<typeof schema>` reads those flags back out. Because presence and nullability are encoded as method-driven flags rather than derived from a declarative shape, the inferred type is sensitive to *which* methods you called and *in what order* — the source of most of Yup's TypeScript friction. Custom methods are added globally via `addMethod` plus `declare module 'yup'` interface merging, which the docs themselves flag as fragile (the augmentation must match the internal generics exactly). As of v1, Yup also implements the [Standard Schema](https://github.com/standard-schema/standard-schema) interface, so it can be consumed by any library that speaks that spec.

## Production Notes

- **Type inference is the main footgun.** `strictNullChecks` (or `strict`) *must* be enabled in `tsconfig.json` or inference silently produces wrong types[^4]. `.required()` on a string does not exclude `null` the way you might expect (it means "defined and non-empty"); use `.defined()` / `.nonNullable()` for precise typing. Expect to annotate `mixed()`, `.oneOf([...] as const)`, and `.test()` callbacks manually.
- **`validate()` is async by default.** It returns a `Promise` and, on failure, *rejects* with a `ValidationError` rather than returning a result object. Forgetting to `await` or wrap in try/catch is a common bug. Use `validateSync()` only if no async tests are present.
- **`abortEarly` defaults to `true`** — validation stops at the first failure. For forms you almost always want `{ abortEarly: false }` to collect all errors into `error.inner`.
- **Coercion can hide bugs.** Because casting is on by default, `"24"`, `"true"`, and loosely-formatted dates pass validation as their coerced forms. If you want validation to reject mistyped input, pass `strict: true` — otherwise Yup is a parser, not a gatekeeper.
- **Performance and bundle size** trail newer validators. Yup's per-validation overhead and shipped size are generally higher than Zod's and much higher than Valibot's; for hot paths or bundle-sensitive frontends this is a real consideration. There are no official numbers here — benchmark against your own workload.
- **Migration cost is real.** Pre-v1 (0.x) Yup was JavaScript-first with different defaults and no meaningful inference; the v1 upgrade (2023) changed typing behavior and some runtime semantics. Codebases pinned to 0.32.x should budget for the jump.

## When to Use / When Not

**Use when:**
- You need coercion/parsing as much as validation — turning strings from forms or JSON into typed values.
- You're already on Formik, or in a codebase that standardized on Yup years ago.
- You need expressive interdependent field rules (`ref`, `.when`, conditional schemas) and value them over pristine type inference.
- You want async validation as a first-class concept (e.g. server-side uniqueness checks).

**Avoid when:**
- You're starting a new TypeScript-first project and want inference that "just works" — Zod is the safer default.
- Bundle size or validation throughput is critical — look at Valibot or Zod.
- You want validation that rejects mistyped input by default rather than silently coercing it.
- You need a battle-tested JSON Schema pipeline — Ajv fits that better.

## Alternatives

- colinhacks/zod — inference-first TypeScript validator; the current default for new TS projects. Use instead when clean static types matter more than coercion.
- fabian-hiller/valibot — modular, tree-shakeable validator with a tiny bundle footprint. Use when frontend bundle size is the priority.
- hapijs/joi — mature, coercion-heavy validator for Node backends with no TypeScript inference story. Use for server-side validation where types don't matter.
- ajv-validator/ajv — JSON Schema validator, fastest at runtime. Use when you need standards-based JSON Schema or maximum throughput.
- ianstormtaylor/superstruct — small, composable, functional validator. Use when you want a minimal, unopinionated core.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014–2022 | Original JavaScript-first releases; minimal TypeScript inference[^1]. |
| 1.0.0-beta.0 | 2021-12-29 | Start of the TypeScript-focused v1 line. |
| 1.0.0 | 2023-02-08 | Stable v1: `InferType`, reworked typing, generic flag system[^3]. |
| 1.x | 2023–2026 | Standard Schema support, tuple type, incremental fixes (latest 1.7.0). |

## References

[^1]: `jquense/yup` repository, created 2014-09-22. https://github.com/jquense/yup
[^2]: Formik documents Yup as its validation companion; both are authored by Jason Quense (jquense). https://formik.org/docs/guides/validation
[^3]: Yup v1.0.0 release, published 2023-02-08. https://github.com/jquense/yup/releases/tag/v1.0.0
[^4]: Yup README, "TypeScript configuration" — `strictNullChecks` is required for inference to work. https://github.com/jquense/yup#typescript-configuration

## Tags

typescript, javascript, validation, schema-validation, runtime-validation, type-inference, forms, coercion, standard-schema, npm
