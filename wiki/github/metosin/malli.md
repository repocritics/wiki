# metosin/malli

> Data-driven schemas for Clojure/Script — validation, coercion, generation, and JSON Schema, with schemas that are plain data you can serialize and edit at runtime.

[GitHub repo](https://github.com/metosin/malli) ·
[Online demo](https://malli.io) ·
[License: EPL-2.0](https://github.com/metosin/malli/blob/master/LICENSE)

## Overview

Malli is a schema library for Clojure, ClojureScript, and babashka, built by Metosin, the Finnish consultancy behind reitit and spec-tools. It started in 2019 as an explicit reaction to the two incumbents: Plumatic Schema (hard to serialize) and clojure.spec (macro-based, global registry, no runtime transformation support)[^1]. Malli's founding requirement was dynamic multi-tenant systems where data models are first-class values — editable at runtime, persisted to a database, sent over the wire — so every schema is plain data (a hiccup-style vector like `[:map [:x :int]]`) rather than a macro expansion or opaque object.

Beyond validation, the library bundles what would be five separate libraries elsewhere: humanized error messages with spell checking, bidirectional value transformation (string/JSON coercion), test.check-based value generation, schema inference from sample data, JSON Schema / Swagger2 output, and function schemas with dev-time instrumentation plus clj-kondo static-check integration[^2].

The defining tension is stability semantics. Malli self-describes as "well matured *alpha*": widely used in production (reitit coercion, the wider Metosin stack, babashka builds it in), actively maintained (last push 2026-05; ~1.7k stars, 237 forks), yet minor releases still ship deliberate breaking changes — 0.16 raised the Java floor, 0.18 changed `parse` output, 0.19 changed JSON encoding of `:enum`[^3]. It is production-grade software with pre-1.0 contract discipline, and users are expected to read changelogs.

## Getting Started

```clojure
;; deps.edn
{:deps {metosin/malli {:mvn/version "0.20.1"}}}
```

Requires Clojure 1.11+ / ClojureScript 1.11.51+; tested on JDK 8–25.

```clojure
(require '[malli.core :as m]
         '[malli.error :as me]
         '[malli.transform :as mt])

(def Address [:map [:street :string] [:zip :int] [:country [:enum "FI" "UA"]]])

(m/validate Address {:street "Hämeenkatu 1" :zip 33100 :country "FI"}) ;; => true

(-> Address (m/explain {:street "x" :zip "33100" :country "SE"}) (me/humanize)) ;; => {:zip ["should be an integer"], :country ["should be either FI or UA"]}

;; coerce string input (e.g. query params) into the schema's types
(m/decode Address {:street "x" :zip "33100" :country "FI"} mt/string-transformer)
;; => {:street "x", :zip 33100, :country "FI"}
```

## Architecture / How It Works

Schemas exist in three syntaxes: **vector** (the hiccup-inspired default), **map/AST** (parseless creation for slow single-threaded targets like mobile JS — documented as internal, not a persistence format), and **lite** (flat shorthand for reitit coercion and data-specs migration)[^2]. All compile via `m/schema` into objects implementing the `Schema` protocol; `m/form` round-trips back to data.

The performance model is compile-once/call-many: `m/validator`, `m/explainer`, `m/parser`, and `m/decoder` return precompiled pure functions. The convenience arities (`m/validate` on a raw form) recompile through a cache and are markedly slower in hot paths. Metosin's published microbenchmarks show the compiled sequence-regex engine (`:cat`/`:alt`/`:*`/`:repeat`, a custom seqex implementation) running roughly an order of magnitude faster than the equivalent clojure.spec forms[^4].

Schema types resolve through **registries**. The default is an immutable map of type → schema; you can also attach local registries as schema properties, or use mutable, dynamic, and lazy registries for runtime-loaded models. Swapping the default registry requires a compile-time flag (`-Dmalli.registry/type=custom` on the JVM, a compiler option in CLJS) — a deliberate hurdle, since the default registry is baked in for performance and dead-code elimination reasons[^5].

Other notable subsystems: **transformers** are composable, bidirectional (decode/encode) interceptor chains — `string-transformer`, `json-transformer`, `strip-extra-keys-transformer`, `default-value-transformer` stack with `mt/transformer`. **Function schemas** (`:=>`, and the flat `:->` since 0.16.3) attach to vars via `m/=>`; `malli.instrument` rewrites vars for runtime checking, `malli.dev/start!` adds file-watch re-instrumentation and pretty errors, and `malli.clj-kondo/emit!` exports static type configs. **Serializable functions** in persisted schemas are evaluated through the embedded SCI interpreter (optional dependency — without it, `m/eval` is disabled).

## Production Notes

**Breaking changes arrive in minor versions.** Recent examples: 0.13 changed `:or` decoding semantics; 0.16.0 required Java 11 (0.16.1 restored Java 8); 0.17 made `:gen/fmap` behavior stricter; 0.18 replaced `parse` output for `:orn`/`:catn`/`:multi` with new `Tag`/`Tags` record types — a silent shape change for any code destructuring parse results; 0.19 changed how `json-transformer` encodes `:enum`/`:=` values[^3]. Pin the version and read the changelog before every bump.

**Compile schemas at load time.** The single most common performance mistake is calling `m/validate`/`m/decode` with a vector form inside a request loop. Derive `m/validator` / `m/decoder` once (a `defn` closing over them, or a top-level `def`) — the difference is easily 10x+ in hot paths[^4].

**Maps are open by default.** `[:map [:x :int]]` accepts any extra keys. For API boundaries this is a footgun: use `{:closed true}`, `mu/closed-schema`, or `strip-extra-keys-transformer` on decode, or unexpected fields flow straight through validation into your system.

**ClojureScript bundle size.** The default registry references every built-in schema, which defeats dead-code elimination; SCI adds more. Shrinking a production JS bundle means opting into a custom default registry via the compile-time flag and registering only the schema types you use[^5].

**Mutable registries are shared global state.** The project itself documents the caveats (0.20.1 added a dedicated section)[^6]: load-order sensitivity and cross-library collisions are real; prefer local or composite registries in libraries, and reserve the mutable registry for application top-level.

**Instrumentation is a dev-time tool.** `malli.dev/start!` wraps vars with checking overhead and captures thrown exceptions for pretty printing — do not ship it enabled. The supported production-adjacent path is clj-kondo static checks in CI plus selective runtime asserts at boundaries.

**Maintenance profile.** Single-company stewardship (Metosin, "Active" status on their open-source ladder[^7]) with releases roughly quarterly; ~140 open issues/PRs. Realistic expectation: responsive but paced by consultancy priorities, with commercial support available.

## When to Use / When Not

**Use when:**
- Schemas must be data: stored in a DB, edited at runtime, sent over the wire, or shared between Clojure and ClojureScript.
- You want validation, coercion, generation, and JSON Schema/Swagger output from one schema definition (e.g. reitit HTTP coercion).
- You need fast compiled validators in hot request paths.
- You run babashka or GraalVM native images — malli is compatible with both.

**Avoid when:**
- You need frozen API stability — "matured alpha" means minors can break you.
- Your team is already invested in clojure.spec and needs none of the runtime transformation/serialization features; the migration cost buys little.
- You want static type checking as the primary safety net — malli's clj-kondo and Typed Clojure integrations annotate, they don't replace a type system.
- You are outside the JVM/JS Clojure ecosystem entirely; this is not a general-purpose validation library.

## Alternatives

- plumatic/schema — the proven predecessor; use it when you want maximal stability and don't need schemas as serializable data.
- clojure/spec.alpha — bundled with Clojure; use it for idiomatic in-language specs, global registry, and generative testing without runtime coercion needs.
- metosin/spec-tools — spec plus runtime transformation, by the same authors; effectively superseded by malli, relevant only for existing spec codebases.
- fulcrologic/guardrails — use it when function-signature checking with a gspec-style `>defn` syntax is the primary goal rather than data schemas.
- exoscale/coax — spec-based coercion; use it to add coercion to an existing spec investment instead of switching schema libraries.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-05 | Repo created; ~18 months of pre-release development[^1]. |
| 0.2.0 | 2020-10 | First minor release line on Clojars. |
| 0.8.0 | 2022-01 | Function schema instrumentation, `malli.dev` tooling, clj-kondo integration. |
| 0.13.0 | 2023-09 | Breaking `:or`/`:orn` decode semantics; faster `:map` generators. |
| 0.14.0 | 2024-01 | `malli.dev/start!` captures all malli exceptions; pretty-printer API break. |
| 0.16.0 | 2024-04 | Breaking: Java 11 minimum (Java 8 restored in 0.16.1); `:->` arrives 0.16.2–0.16.3. |
| 0.17.0 | 2024-12 | `json-transformer` keyword-key decoding; stricter `:gen/fmap`. |
| 0.18.0 | 2025-05 | Breaking: `parse` output uses `Tag`/`Tags` records[^3]. |
| 0.19.0 | 2025-06 | Breaking: `json-transformer` encoder inference for `:enum`/`:=`. |
| 0.20.0 | 2025-11 | Robust `:and` parser, `:andn`, recursive-schema fixes, JDK 25 CI. |
| 0.20.1 | 2026-03 | Experimental `:validate` schema for custom errors; mutable-registry docs. |

## References

[^1]: Malli README, "Motivation" — rationale vs. Schema, spec, and spec-tools. https://github.com/metosin/malli#motivation
[^2]: Malli README — feature list and syntax documentation. https://github.com/metosin/malli
[^3]: Malli releases/CHANGELOG — 0.16.0, 0.18.0, 0.19.0 breaking-change entries. https://github.com/metosin/malli/releases
[^4]: Metosin blog, "Structure and Interpretation of Malli Regex Schemas" — seqex engine design and benchmarks vs. spec. https://www.metosin.fi/blog/malli-regex-schemas/
[^5]: Malli README, "Schema Registry" — custom default registry and `malli.registry/type=custom` compile flag. https://github.com/metosin/malli#schema-registry
[^6]: Malli 0.20.1 release notes — mutable registry caveats documentation. https://github.com/metosin/malli/releases/tag/0.20.1
[^7]: Metosin open-source project status ladder. https://github.com/metosin/open-source/blob/main/project-status.md

## Tags

clojure, clojurescript, schema, validation, coercion, data-driven, json-schema, generative-testing, babashka, specification 