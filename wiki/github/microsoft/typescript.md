# microsoft/typescript

> TypeScript is a typed superset of JavaScript that compiles to plain JavaScript.

[GitHub repo](https://github.com/microsoft/typescript) ·
[Official website](https://www.typescriptlang.org) ·
[License: Apache-2.0](https://github.com/microsoft/TypeScript/blob/main/LICENSE.txt)

## Overview

TypeScript is a statically typed superset of JavaScript, designed by Anders Hejlsberg (Turbo Pascal, Delphi, C#) and released by Microsoft in 2012[^1]. It adds a structural type system, generics, type inference, JSX support, and an ahead-of-time compiler (`tsc`) that emits plain JavaScript. The Language Service that powers IDE intellisense is the single most consequential piece of the project — it is what made TypeScript the dominant statically-typed JS layer despite Flow having shipped concurrently.

By 2026 TypeScript is the default language for new JavaScript projects at most companies. Most real-world TS pipelines no longer use `tsc` for transpilation — that work is delegated to `esbuild`, `swc`, or Babel for speed — but `tsc --noEmit` remains the canonical type checker. The compiler is itself written in TypeScript, which has historically capped its performance ceiling.

The most significant architectural change since 2012 was announced in 2025: a **Go port of the compiler** ("TypeScript Native") targeting roughly 10× speedups on large projects[^2]. As of this writing it is in preview, parallel to the existing TS-self-hosted compiler.

## Getting Started

```bash
npm install -D typescript
npx tsc --init
```

`tsconfig.json` (modern baseline):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "noEmit": true
  }
}
```

A minimal typed program:

```ts
type User = { id: string; name: string; email?: string }

function greet({ name }: User): string {
  return `Hello, ${name}`
}

const u: User = { id: "1", name: "Tom" }
console.log(greet(u))
```

Type-check without emitting (most projects):

```bash
npx tsc --noEmit
```

## Architecture / How It Works

TypeScript's type system is **structural**: two types are compatible if their shapes match, regardless of name. This differs from nominal type systems (Java, C#) and is closer to OCaml's row polymorphism. Inference is bidirectional with contextual typing — function parameters take their types from the call site when no annotation exists.

The compiler pipeline:
1. **Scanner / Parser** — produces an AST.
2. **Binder** — creates symbol tables for scope resolution.
3. **Checker** — the largest single module; performs type inference, generic instantiation, assignability checks.
4. **Emitter** — strips types, transforms newer syntax to the configured `target`, emits JS.

**Generic type system** includes conditional types (`T extends U ? X : Y`), mapped types (`{ [K in keyof T]: ... }`), template literal types (`` `${T}-${U}` ``), and recursive type aliases. Together these form a Turing-complete type-level language[^3] — useful for library authors, dangerous in application code where it tanks compile times.

**No runtime.** TypeScript types are fully erased at emit. Runtime type information requires schema libraries (`zod`, `valibot`, `effect/schema`) or decorators (with metadata reflection, niche). This is the single most common source of confusion for developers from Java/C# backgrounds.

**The TypeScript Native (Go port)** rewrites the compiler in Go while keeping the existing TypeScript-authored compiler as the spec[^2]. The goal is single-digit-second type checks on million-line projects. Initial benchmarks at announcement showed 10× speedup; production parity is the gating release criterion.

## Production Notes

**Performance at scale.** `tsc` performance is O(complex) in type complexity. The largest costs:
- Deep conditional types and recursive mapped types.
- `unknown` / `any` propagation forcing repeated re-inference.
- Inheriting types from large `node_modules` packages (`skipLibCheck: true` helps).
- Project references (`composite: true` + `references`) are the official answer for monorepos — they enable incremental builds.

**`isolatedModules` + `verbatimModuleSyntax`.** Required for esbuild / swc / Babel compatibility. Forces explicit `import type` for type-only imports and disallows certain const-enum patterns. Strongly recommended for new projects.

**Strict mode.** `strict: true` enables a bundle of checks (`noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, etc.). New projects should always enable it. Legacy migration is best done one flag at a time, starting with `noImplicitAny`.

**`any` and `unknown`.** `any` opts out of the type system entirely; `unknown` is type-safe by forcing narrowing before use. New code should never introduce `any`. Linter rules: `@typescript-eslint/no-explicit-any`.

**Decorators.** ES decorators (TC39 Stage 3) ship in TS 5.0+ but differ from the legacy `experimentalDecorators` semantics used by Angular, NestJS, TypeORM. Migration to stage-3 decorators is incomplete across the ecosystem — Angular shipped its own transition plan, others lag.

**Editor performance.** The TS language service runs in your editor. Large projects (~100k LOC+) regularly produce 5–20 second hover/completion latency. Mitigations: project references, `skipLibCheck`, smaller per-package surface areas in monorepos.

## When to Use / When Not

**Use when:**
- You're starting a JavaScript project of any non-trivial size — TypeScript is the default in 2026.
- You're building a library that will be consumed by other TS projects (your `.d.ts` is your API contract).
- You're working in a team and need API contracts that survive refactors.

**Avoid when:**
- You're writing a 50-line script. Plain JS + JSDoc types is faster.
- You're targeting a constrained runtime that can't tolerate the build step (rare in 2026).
- You're prototyping and the type ceremony is slowing exploration — use `any` liberally, tighten later.

## Alternatives

- **JSDoc types** — type-check JavaScript with `// @ts-check` and JSDoc annotations. Same checker, no compile step. Good for libraries that want to ship JS without TS as a dep.
- **Flow** — Facebook's competitor, effectively dead as of 2024 outside Meta itself.
- **Hegel, Sorbet (Ruby), mypy (Python)** — type checkers for other languages, not direct competitors.
- TypeScript Native (Go port) is **not** an alternative — it's the same language, faster implementation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.8 | 2012-10 | Initial public release[^1]. |
| 1.0 | 2014-04 | Stable, generics. |
| 1.6 | 2015-09 | JSX support, intersection types. |
| 2.0 | 2016-09 | Strict null checks, control flow analysis. |
| 2.8 | 2018-03 | Conditional types. |
| 3.0 | 2018-07 | Project references, tuples in rest params. |
| 4.0 | 2020-08 | Variadic tuples, labeled tuple elements. |
| 4.1 | 2020-11 | Template literal types[^3]. |
| 5.0 | 2023-03 | ES decorators, `const` type parameters, smaller package. |
| 5.4 | 2024-03 | `NoInfer`, narrowing improvements. |
| 5.5 | 2024-06 | Inferred type predicates. |
| Native | 2025+ | Go port preview[^2]. |

## References

[^1]: Anders Hejlsberg et al., "Announcing TypeScript 1.0" — 2014. https://devblogs.microsoft.com/typescript/announcing-typescript-1-0/
[^2]: Anders Hejlsberg, "A 10x Faster TypeScript" — 2025-03-11. https://devblogs.microsoft.com/typescript/typescript-native-port/
[^3]: Anders Hejlsberg, "Template Literal Types" — TypeScript 4.1. https://devblogs.microsoft.com/typescript/announcing-typescript-4-1/
