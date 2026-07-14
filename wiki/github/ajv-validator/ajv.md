# ajv-validator/ajv

> A JSON Schema validator that compiles schemas into specialized JavaScript validation functions — the de facto validation engine underneath much of the Node.js ecosystem.

[GitHub repo](https://github.com/ajv-validator/ajv) ·
[Official website](https://ajv.js.org) ·
[License: MIT](https://github.com/ajv-validator/ajv/blob/master/LICENSE)

## Overview

Ajv ("Another JSON Validator") validates data against JSON Schema. Its
defining design choice is code generation: rather than interpreting a schema
at validation time, `ajv.compile(schema)` emits a purpose-built JavaScript
function for that exact schema, which the V8 engine can then JIT-optimize.
This is why Ajv consistently tops JSON Schema validation benchmarks[^1] and
why it became the default validator inside tools like Fastify, ESLint's config
loader, webpack schema checks, and countless API frameworks.

Ajv supports JSON Schema drafts 06, 07, 2019-09, and 2020-12, plus JSON Type
Definition (RFC 8927)[^2]. Draft-04 support requires the separate
`ajv-draft-04` package. The project is maintained primarily by its original
author, Evgeny Poberezkin (epoberezkin), and has received a Mozilla Open
Source Support (MOSS) grant[^3]. With roughly 14.8k stars but over 100M weekly
npm downloads, it is a case where GitHub stars badly understate reach — Ajv is
infrastructure, pulled in transitively far more often than it is chosen
directly.

The central tension: code generation is what makes Ajv fast, and also what
makes it awkward in Content-Security-Policy environments (it uses `Function`
constructor / eval semantics), risky when compiling untrusted schemas, and the
source of a difficult v6→v7/v8 migration that split the ecosystem.

## Getting Started

```bash
npm install ajv
```

```javascript
import Ajv from "ajv"

const ajv = new Ajv({ allErrors: true })

const schema = {
  type: "object",
  properties: {
    foo: { type: "integer" },
    bar: { type: "string" },
  },
  required: ["foo"],
  additionalProperties: false,
}

const validate = ajv.compile(schema)   // compile ONCE, reuse the function
if (!validate({ foo: 1, bar: "abc" })) {
  console.log(validate.errors)
}
```

Formats (`date`, `email`, `uri`, etc.) are not built in since v7 — install and
register `ajv-formats` separately:

```javascript
import addFormats from "ajv-formats"
addFormats(ajv)
```

## Architecture / How It Works

`compile()` walks the schema tree and, for each keyword, appends JavaScript
source to a code buffer using an internal `CodeGen` API. The result is a single
function containing inlined checks (`if (typeof data.foo !== "number") ...`),
error-collection logic, and any referenced sub-validators. That function is
instantiated once and called on every payload — validation itself does no
schema traversal, which is the entire performance story.

Ajv v6 built this generated code from `doT.js` string templates. v7 replaced
that with a typed `CodeGen` module that produces an AST-like intermediate form
before serialization — a rewrite that improved tree-shaking, TypeScript types,
and the standalone-compilation path, but changed enough internals to make v6
plugins incompatible[^4].

Two features that flow from the code-gen model:

- **Standalone validation code** — `ajv.compile` output can be serialized to a
  standalone `.js` module (via `ajv/dist/standalone`) at build time, so the
  runtime never calls `eval`/`Function` at all. This is the sanctioned escape
  hatch for CSP-restricted and edge/serverless environments.
- **Data mutation** — because the validator is generated per-schema, Ajv can
  cheaply fold in behaviors like `removeAdditional`, `useDefaults`, and
  `coerceTypes` that mutate the input during validation. Convenient, but it
  means validation is not always side-effect-free.

References (`$ref`, `$recursiveRef`, `$dynamicRef`) are resolved at compile
time; remote schemas must be registered via `addSchema` or loaded through the
async `loadSchema` option before/while compiling.

## Production Notes

- **Compile once.** The most common performance bug is calling
  `ajv.compile()` (or `ajv.validate(schema, data)`, which compiles internally)
  on every request. Compilation is expensive; the compiled function is cheap.
  Cache compiled validators, or use `addSchema` + `getSchema(key)`.

- **CSP / `unsafe-eval`.** Default Ajv generates code with the `Function`
  constructor. Under a strict Content-Security-Policy (no `unsafe-eval`),
  compilation throws. Use the standalone precompilation path at build time
  rather than shipping schemas to the browser and compiling there.

- **Strict mode is on by default (v7+).** Unknown keywords, unknown formats,
  ambiguous `additionalProperties`/`required` combinations, and several other
  patterns now throw at compile time instead of being ignored. This is the
  single biggest source of "it worked in v6" migration pain. `strict: false`
  restores lenient behavior but masks real schema mistakes.

- **The two-major split.** Ajv 6 and Ajv 8 are incompatible and frequently
  coexist in the same `node_modules` tree because widely-used tools pinned v6
  (older ESLint, webpack, and their plugins). Expect duplicate copies; a schema
  and plugin must be matched to the same major (e.g. `ajv-formats` requires
  v8, `ajv-keywords` versions are major-specific).

- **Untrusted schemas are a code-execution surface.** Compiling
  attacker-controlled schemas means generating and running attacker-influenced
  code, and format regexes can enable ReDoS. Ajv has had prototype-pollution
  and ReDoS advisories in its history[^5]. Validate *data* from untrusted
  sources freely; do not `compile` *schemas* from untrusted sources.

- **`additionalProperties: false` with `allOf`/`$ref` is a footgun** — it does
  not "see through" combinators, so a property allowed by a referenced schema
  is still rejected. This is JSON Schema semantics, not an Ajv bug, but Ajv is
  where people hit it.

- **`allErrors: true` is slower and a DoS vector** on hostile input (it forces
  full traversal instead of failing fast); keep it off in hot paths.

## When to Use / When Not

**Use when:**
- You need standards-compliant JSON Schema validation (OpenAPI, config files,
  API request/response bodies against a spec you don't own).
- Validation is hot and you can compile schemas ahead of time.
- You want interop: schemas are portable data, not code, and work across
  languages and tools.

**Avoid / reconsider when:**
- Your "schema" is really TypeScript types — `zod` or `typebox` give you types
  and validation from one source without hand-writing JSON Schema.
- You run under strict CSP and cannot use the standalone build.
- You must compile schemas supplied by end users at runtime (code-gen risk).
- You only need a couple of shape checks — Ajv's compile cost and API surface
  may be overkill for a trivial guard.

## Alternatives

- colinhacks/zod — TypeScript-first; schema and static type are one definition. Use when your schemas live in code, not as portable JSON.
- sinclairzx81/typebox — builds JSON Schema *from* TS types; frequently paired *with* Ajv rather than replacing it. Use when you want both TS types and a real JSON Schema to hand to Ajv.
- ExodusMovement/schemasafe — security-focused JSON Schema validator with an eval-free mode. Use when compiling untrusted schemas or under strict CSP.
- cfworker/json-schema — small, no `eval`, edge-runtime friendly. Use on Cloudflare Workers / CSP-locked environments where Ajv's code-gen is a problem.
- hapijs/joi — object-schema validation with a fluent JS API, not JSON Schema. Use when you want ergonomic in-code validation and don't need spec portability.

## History

| Version | Date | Notes |
|---------|------|-------|
| 4.x | 2016 | Early widely-adopted line; draft-04. |
| 6.0.0 | 2017-12 | draft-07 support; the long-lived version many tools still pin[^4]. |
| 7.0.0 | 2020-12 | CodeGen rewrite (dropped doT templates), strict mode default, formats split into `ajv-formats`[^4]. |
| 8.0.0 | 2021-03 | Added JSON Schema 2019-09 / 2020-12 and JSON Type Definition (RFC 8927)[^2]. |

Current development targets the v8 line; the README notes further major work is
planned pending sponsorship. See the GitHub releases page for the full
changelog[^6].

## References

[^1]: json-schema-benchmark, comparing JSON Schema validators (Ajv reported fastest). https://github.com/ebdrup/json-schema-benchmark
[^2]: JSON Type Definition, RFC 8927. https://datatracker.ietf.org/doc/rfc8927/
[^3]: Ajv README — Mozilla Open Source Support (MOSS) grant acknowledgment. https://github.com/ajv-validator/ajv
[^4]: Ajv release notes for v7.0.0 (CodeGen rewrite, strict mode, formats moved out). https://github.com/ajv-validator/ajv/releases/tag/v7.0.0
[^5]: Ajv security policy and advisory history (report via Tidelift). https://ajv.js.org/security.html
[^6]: Ajv releases. https://github.com/ajv-validator/ajv/releases

## Tags

typescript, javascript, json-schema, validation, json-type-definition, code-generation, nodejs, schema-validator, openapi, data-validation
