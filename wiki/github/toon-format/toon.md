# toon-format/toon

A compact, human-readable encoding of the JSON data model designed to cut token usage when feeding structured data to LLMs.

## What it is

Token-Oriented Object Notation (TOON) is a lossless, drop-in representation of JSON intended specifically for LLM input. It combines YAML-style indentation for nested objects with a CSV-style tabular layout for uniform arrays of objects, declaring fields once and streaming row values line by line. The problem it addresses is that standard JSON is verbose and token-expensive; TOON keeps the same data model while using fewer tokens and adding explicit structure ([N] lengths and {fields} headers) that helps models parse and validate the data. It is aimed at developers building LLM prompts who work with JSON programmatically and want a cheaper on-the-wire encoding for model input. The repository holds the spec, benchmarks, and a TypeScript SDK.

## Key features

- Lossless, deterministic round-trips of the JSON data model (objects, arrays, primitives).
- Tabular arrays: uniform arrays of objects collapse into tables with a single field header and one row per item.
- Explicit `[N]` length markers and `{fields}` headers that give models a schema to follow and enable structural validation.
- Minimal syntax using indentation instead of braces and reduced quoting, for YAML-like readability with CSV-like compactness.
- A published benchmark suite measuring retrieval accuracy and token efficiency across multiple models and data shapes.
- A TypeScript library with `encode`, streaming `encodeLines`, and streaming decode APIs, plus a CLI for JSON↔TOON conversion.

## Tech stack

- Primary language: TypeScript (`typescript ^6.0.3`).
- A pnpm monorepo (`pnpm@11.13.0`) with workspace packages `@toon-format/toon` (the library) and `@toon-format/cli`.
- Build via `tsdown ^0.22.8`; tests via `vitest ^4.1.10`; linting via ESLint with `@antfu/eslint-config`.
- `@types/node ^26.1.1`; documentation built with VitePress and deployed via Cloudflare (`wrangler`).
- Spec-driven implementations are noted in TypeScript, Python, Go, Rust, and .NET, among other languages.

## When to reach for it

- Passing uniform arrays of objects to an LLM where token cost matters and the structure is regular.
- Wanting a lossless JSON↔TOON translation layer: keep JSON in code, encode as TOON only for model input.
- Wanting explicit length and field-header guardrails to improve an LLM's parsing and validation reliability.
- Streaming large uniform datasets into a prompt with a memory-efficient line-by-line encoder.

## When *not* to reach for it

- Deeply nested or non-uniform structures, where the README notes compact JSON often uses fewer tokens.
- Semi-uniform arrays (roughly 40–60% tabular eligibility), where the token savings diminish.
- Pure flat tabular data, where CSV is smaller; TOON adds ~5–10% overhead for its structural markers.
- Latency-critical setups, especially local or quantized models, which the README says may process compact JSON faster despite TOON's lower token count — it advises benchmarking your exact setup.

## Maturity signal

TOON drew a large following quickly, passing 24,000 stars within roughly nine months of its creation, and it is actively maintained with very recent pushes and an ongoing spec (v3.3). It is MIT-licensed and organized as a versioned monorepo. The maintainers describe the format as stable but still "an idea in progress" and openly invite spec feedback. Treat it as young but rapidly adopted and actively developed, with a format that may continue to evolve.

## Alternatives

- Compact (minified) JSON — use it when data is deeply nested or non-uniform, where the benchmarks show it can beat TOON on token count.
- CSV — use it for purely flat tabular data, where it is smaller than TOON, if you do not need structural validation or nesting.
- YAML — use it when human readability and hand-editing matter more than minimizing tokens.

## Notes

The benchmarks are unusually candid about where TOON loses: on the "array truncated" validation case TOON scored 0% while CSV and XML scored 100%, and compact JSON wins on token count for nested and semi-uniform data. TOON's reported edge is accuracy-per-token — 76.4% accuracy versus JSON's 75.0% while using about 40% fewer tokens on mixed-structure datasets. Files use the `.toon` extension and a provisional `text/toon` media type, and documents are always UTF-8. The benchmark measures models reading TOON, not generating it.

## Tags

typescript, data-format, serialization, llm, tokenization, cli, library, json, monorepo, spec
