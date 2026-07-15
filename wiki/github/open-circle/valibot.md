# open-circle/valibot

> A modular, tree-shakable TypeScript schema library — the same job as Zod, optimized for bundle size instead of API ergonomics.

[GitHub repo](https://github.com/open-circle/valibot) ·
[Official website](https://valibot.dev) ·
[License: MIT](https://github.com/open-circle/valibot/blob/main/LICENSE.md)

## Overview

Valibot is a runtime schema validation library for TypeScript: you declare a schema, validate unknown data against it at runtime, and infer a static type from the same declaration. It occupies the same niche as Zod — parse-don't-validate at trust boundaries (HTTP payloads, form input, env vars, config files) with a single source of truth for the type and the check[^1].

The defining decision is a **fully modular, function-based API**. Where Zod uses method chaining (`z.string().email().min(8)`), Valibot composes free-standing functions through a `pipe` (`v.pipe(v.string(), v.email(), v.minLength(8))`). Every validator, transform, and schema is a separate top-level export. Because nothing hangs off a shared class, a bundler's tree-shaking can drop every function you don't import — the project claims reductions of up to 95% versus Zod for a used subset, with schemas starting under 1 KB[^2]. The tradeoff is verbosity: constraints that are one method call in Zod are separate imports wrapped in a `pipe` here.

Valibot began as Fabian Hiller's bachelor's thesis at Stuttgart Media University, supervised in part by Miško Hevery (Angular, Qwik) and Ryan Carniato (SolidJS), and was announced in mid-2023[^3]. Its author is also a co-designer of **Standard Schema**, a shared interface (`~standard`) implemented by Valibot, Zod, and ArkType so that downstream libraries can accept any of them without an adapter[^4]. The repository now lives under the `open-circle` organization (transferred from the original `fabian-hiller/valibot`); GitHub redirects the old path.

## Getting Started

```bash
npm install valibot
# also published on JSR as @valibot/valibot
```

```ts
import * as v from 'valibot';

const LoginSchema = v.object({
  email: v.pipe(v.string(), v.email()),
  password: v.pipe(v.string(), v.minLength(8)),
});

// Static type inferred from the schema:
// { email: string; password: string }
type LoginData = v.InferOutput<typeof LoginSchema>;

// Throws a ValiError on failure:
const data = v.parse(LoginSchema, { email: 'jane@example.com', password: '12345678' });

// Non-throwing variant returns a result union:
const result = v.safeParse(LoginSchema, unknownInput);
if (result.success) {
  result.output; // typed LoginData
} else {
  result.issues; // structured issue list
}
```

`is(schema, input)` provides a boolean type guard for cases where you don't need issue details.

## Architecture / How It Works

A Valibot schema is a plain object carrying metadata plus an internal parsing function; it is not an instance of a base class. This is what makes the library tree-shakable — there is no shared prototype dragging every method into the bundle.

Composition happens through `pipe`. `v.pipe(baseSchema, ...actions)` returns a new schema whose parse function runs the actions in order. Actions come in two kinds: **validation actions** (`email`, `minLength`, `integer`) that inspect a value and may emit an issue, and **transformation actions** (`trim`, `transform`, `toLowerCase`) that change the value or its type. Because transforms can change the type mid-pipe, Valibot distinguishes `InferInput` (the accepted input type) from `InferOutput` (the type after transforms) — a distinction that matters whenever a schema coerces or reshapes data.

Everything is a standalone export: schemas (`string`, `object`, `array`, `union`, `record`), actions (the pipe items), and utilities (`parse`, `safeParse`, `is`, `pipe`, `fallback`, `getDefaults`). There is no fluent builder, so IDE discoverability is worse than a chained API — you find capabilities through docs and autocomplete on the `v.` namespace rather than by chaining off a value.

The `~standard` property on every schema implements the Standard Schema contract, so form libraries (TanStack Form, React Hook Form via a resolver), ORMs (`drizzle-valibot`), and other consumers can accept a Valibot schema through the same interface they use for Zod or ArkType.

## Production Notes

- **Verbosity is the daily cost.** Every constraint is an import and a `pipe` wrapper. Teams migrating from Zod consistently report the API feels heavier to write and read, even when the bundle savings are real. Budget for it in code review conventions.
- **Bundle-size wins depend on import shape.** `import * as v from 'valibot'` tree-shakes correctly with modern bundlers (esbuild, Rollup, Vite) because the exports are top-level functions — the namespace import does not pull in unused code. Older bundlers or misconfigured `sideEffects` settings can defeat this; verify with a bundle analyzer before assuming the advertised savings.
- **Bundle size, not parse throughput, is the optimization target.** Runtime performance is broadly comparable to peers for typical payloads, but Valibot is not marketed as the fastest validator, and Zod 4 and ArkType have both closed or reversed raw-throughput gaps. If parse speed on large batches dominates your workload, benchmark against your own data rather than trusting either project's numbers.
- **The 0.x era was churn-heavy.** Before 1.0, the API was reshaped repeatedly. The most disruptive change (around v0.31, mid-2024) replaced the previous "pipe-as-array-argument" form with the current `v.pipe(...)` function — a mechanical but wide-reaching migration across every schema in a codebase. Pin your minor version and read release notes if you are on any pre-1.0 range.
- **`InferInput` vs `InferOutput` is a real footgun.** Using the wrong one around transforms (for example, typing a function parameter with the post-transform type when it receives pre-transform input) produces confusing type mismatches. Be deliberate about which side of a transform each type describes.
- **Smaller ecosystem than Zod.** At roughly 8.8k stars versus Zod's tens of thousands, third-party integrations, Stack Overflow answers, and AI-model familiarity are thinner. Standard Schema mitigates this for libraries that adopt it, but you will still find fewer worked examples for edge cases.

## When to Use / When Not

**Use when:**
- Bundle size is a hard constraint — client-heavy apps, edge/serverless functions, embedded widgets, libraries shipping their own validation.
- You are starting fresh and can adopt the functional style as a team convention.
- You want Standard Schema compatibility with a minimal footprint.

**Avoid when:**
- You value terse, chainable, highly discoverable schema code over a few kilobytes.
- You depend on Zod's large ecosystem of plugins, examples, and integrations, or on tooling that only documents Zod.
- Raw parse throughput on large datasets is the deciding factor — measure alternatives first.

## Alternatives

- colinhacks/zod — the default TypeScript validator; chainable API, far larger ecosystem and mindshare. Use when developer ergonomics and integrations outweigh bundle size.
- arktypeio/arktype — schemas written in TypeScript-like string syntax with a heavily optimized runtime. Use when you want top parse performance and type-syntax definitions.
- sinclairzx81/typebox — JSON-Schema-first; schemas are valid JSON Schema usable with Ajv. Use when you need OpenAPI / JSON Schema output, not just TS inference.
- jquense/yup — older object-schema validator with a chainable API. Use only when maintaining existing Yup code; its type inference is weaker.
- gcanti/io-ts — codec-based validation for `fp-ts` codebases. Use when you already work in a functional-programming stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announcement | 2023-07 | Introduced from Fabian Hiller's bachelor thesis; repo created 2023-07-07[^3]. |
| v0.31 (approx.) | 2024 (mid) | Reworked to the `v.pipe(...)` function API, replacing the earlier pipe-array form — largest pre-1.0 migration. |
| v1.0 | 2025 | First stable release; API frozen, semver-stable going forward[^5]. |

## References

[^1]: Valibot documentation — "Introduction". https://valibot.dev/guides/introduction/
[^2]: Valibot README, "Comparison" — modular design and tree-shaking, up to 95% smaller than Zod for a used subset. https://github.com/open-circle/valibot
[^3]: "Introducing Valibot" — Builder.io blog (2023). https://www.builder.io/blog/introducing-valibot
[^4]: Standard Schema — shared `~standard` interface co-authored by the Valibot, Zod, and ArkType maintainers. https://github.com/standard-schema/standard-schema
[^5]: Valibot releases. https://github.com/open-circle/valibot/releases

## Tags

typescript, javascript, schema-validation, type-safety, runtime-validation, bundle-size, tree-shaking, parsing, standard-schema, zod-alternative
