# hapijs/joi

> Schema description language and runtime data validator for JavaScript — the validation layer born inside the hapi ecosystem.

[GitHub repo](https://github.com/hapijs/joi) ·
[Developer portal](https://joi.dev) ·
License: BSD-3-Clause[^1]

## Overview

joi lets you describe the shape of JavaScript data as a chained schema
(`Joi.object({ name: Joi.string().required() })`) and validate values against
it at runtime, producing either a coerced value or a structured error tree. It
was created in 2012 at Walmart Labs as the validation component of the hapi web
framework, authored primarily by Eran Hammer, and is now maintained under
Sideway Inc.[^2] For most of a decade it was the default answer to "how do I
validate request payloads in Node" — hapi wires joi directly into route
validation, and Express/Koa users adopted it by convention.

joi's defining tension in 2026 is that it predates the TypeScript-first era of
validation. It is a runtime-only validator: a joi schema does not produce a
static TypeScript type, so you either duplicate the type by hand or reach for a
codegen helper. This is the single largest reason greenfield TypeScript
projects now start with zod or one of the Standard Schema libraries instead[^3].
joi remains excellent at what it was built for — deep, coercing, message-rich
validation of untrusted input in Node services — but it carries a large,
non-tree-shakeable bundle and an API designed before static inference mattered.

## Getting Started

```bash
npm install joi
```

```js
const Joi = require('joi');

const schema = Joi.object({
  username: Joi.string().alphanum().min(3).max(30).required(),
  email: Joi.string().email().required(),
  age: Joi.number().integer().min(0).max(120),
  role: Joi.string().valid('user', 'admin').default('user'),
}).required();

const { value, error } = schema.validate(
  { username: 'ab', email: 'not-an-email' },
  { abortEarly: false }
);

if (error) {
  // error.details is an array of every failure when abortEarly:false
  console.error(error.details.map(d => d.message));
} else {
  console.log(value); // coerced + defaults applied
}
```

`schema.validate()` never throws — it returns `{ value, error }`. For rules
that need I/O (e.g. `external()`), use `await schema.validateAsync(input)`,
which throws a `ValidationError` on failure.

## Architecture / How It Works

A joi schema is an immutable, chainable object. Every method
(`.min()`, `.required()`, `.valid()`) returns a new frozen schema rather than
mutating in place, so schemas are safe to share and extend. Validation walks
the schema tree against the input, applying type coercion (string→number,
string→date), then rules, then transforms, accumulating failures.

Key internals worth understanding:

- **Coercion by default.** joi will convert types unless you pass
  `{ convert: false }`. `Joi.number()` accepts `"42"` and hands back `42`. This
  is convenient for HTTP payloads but surprising if you expected strict typing.
- **`error.details`.** Every failure is an object with `message`, `path`,
  `type`, and `context`. With `abortEarly: false` you get all of them; the
  default `abortEarly: true` stops at the first error for speed.
- **Refs and `Joi.ref()`.** Cross-field rules (`Joi.ref('password')`,
  `.when()`) let one field's validity depend on another — a feature many
  lighter validators lack or bolt on awkwardly.
- **`Joi.extend()`.** Custom types and rules are added by extending the base,
  which is how the ecosystem builds domain validators.
- **No static types.** The schema exists only at runtime. There is no
  first-class `z.infer` equivalent; `joi-to-typescript` and similar codegen
  tools bridge the gap externally.

Since v17 (2020) joi is a standalone package again. Between roughly v14 and v16
it shipped as `@hapi/joi` under the hapi scope; v17 decoupled it and
re-published as plain `joi`, so the version you install and the import name can
be a migration footgun on older codebases[^4].

## Production Notes

**Bundle size and the browser.** joi is a Node-first library. It is large
(hundreds of KB unminified) and not meaningfully tree-shakeable — importing it
pulls in the whole validator. It runs in browsers via a bundler, but shipping
joi to the client is a common source of oversized bundles; frontend teams
typically pick a smaller validator. Measure before putting it on a critical
client path.

**Coercion surprises.** Because `convert` defaults to `true`, joi silently
transforms input. `Joi.boolean()` accepting `"true"`/`"false"` strings, or
`Joi.number()` swallowing numeric strings, is usually desired for form data but
can mask upstream bugs. Set `{ convert: false }` when you want strict shape
checks.

**abortEarly in APIs.** The default `abortEarly: true` returns only the first
error, which produces poor API responses ("email invalid" while three other
fields are also wrong). Almost every request-validation setup should pass
`abortEarly: false` and map `error.details` to a field-keyed response.

**Performance.** joi is fast enough for typical request validation but is not
the throughput leader. For hot paths validating large volumes against fixed
schemas, ajv (compiled JSON Schema) is materially faster; benchmarks
consistently place compiled validators ahead of joi. Reuse compiled joi schema
instances — never rebuild a schema per request.

**Maintenance cadence.** joi is maintained but no longer moves quickly. The
broader hapi ecosystem went through a well-known 2020 funding episode in which
its lead stepped back from commercial stewardship; development continued but
the pace and community energy shifted toward newer validators. Treat joi as
stable-and-slow rather than fast-moving[^2].

**TypeScript.** The bundled `.d.ts` types cover the API surface but do not
infer a result type from a schema. If your team's value proposition is
"the schema is the source of truth for types," joi will fight you.

## When to Use / When Not

**Use when:**
- You are in the hapi framework, where joi is the native, wired-in validator.
- You need rich cross-field conditional logic (`.when()`, refs) and detailed,
  customizable error messages for untrusted input.
- You are validating server-side payloads in JavaScript (not TypeScript-typed)
  and want a mature, battle-tested API.
- You want coercion of loosely-typed input (query strings, form bodies) built
  in.

**Avoid when:**
- You want the schema to generate your TypeScript types — use zod or valibot.
- Client bundle size matters — joi is heavy and not tree-shakeable.
- You need maximum validation throughput — a compiled JSON Schema validator
  (ajv) will be faster.
- You want an actively fast-iterating project with a large current community.

## Alternatives

- colinhacks/zod — TypeScript-first; the schema infers the static type. The
  default modern choice when you use TypeScript.
- ajv-validator/ajv — compiled JSON Schema validation; use when you need raw
  speed or interoperable JSON Schema documents.
- jquense/yup — lighter, object-schema validator popular with React form
  libraries; use for client-side form validation.
- fabian-hiller/valibot — modular and tree-shakeable with type inference; use
  when bundle size on the client is the priority.
- Standard Schema (standardschema) — a shared interface, not a validator; use
  when you want to stay portable across zod/valibot/arktype.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Created at Walmart Labs as hapi's validation layer[^2]. |
| 6.0 | 2015-03-02 | Established the chainable schema API in wide use today. |
| 11.0 | 2017-09-14 | Ongoing rule/type expansion during peak hapi adoption. |
| 14.0 | 2018-10-14 | Shipped under the `@hapi/joi` scope. |
| 16.0 | 2019-09-11 | Major API rewrite; `validate()` returns `{ value, error }`, callback style removed. |
| 17.0 | 2020-01-04 | Decoupled from hapi scope; republished as standalone `joi`[^4]. |
| 18.0 | 2025-08-03 | Current major line (18.x). |

## References

[^1]: joi LICENSE.md — standard 3-clause BSD text (GitHub reports NOASSERTION due to multi-holder copyright header). https://github.com/hapijs/joi/blob/master/LICENSE.md
[^2]: hapi ecosystem history and Sideway Inc. stewardship; project policies. https://joi.dev/policies/
[^3]: joi API — runtime validation, no static type inference. https://joi.dev/api/
[^4]: joi versions status and changelog (scope change, v17 standalone). https://joi.dev/resources/changelog/

## Tags

javascript, nodejs, validation, schema, data-validation, hapi, runtime-validation, backend, input-validation, library
