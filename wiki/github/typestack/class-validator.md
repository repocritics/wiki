# typestack/class-validator

> Decorator-based property validation for TypeScript/JavaScript classes, wrapping validator.js behind a declarative annotation API.

[GitHub repo](https://github.com/typestack/class-validator) ·
[License: MIT](https://github.com/typestack/class-validator/blob/develop/LICENSE)

## Overview

class-validator lets you attach validation rules to class properties as
decorators (`@IsEmail()`, `@Length(10, 20)`, `@Min(0)`) and validate an
instance at runtime by calling `validate(obj)`. It is part of the typestack
family of decorator-driven libraries (class-transformer, routing-controllers,
typedi) and is best known as the validation layer behind NestJS's
`ValidationPipe`[^1]. For string-level checks it does not reimplement anything —
it delegates to validator.js, the long-standing string-validation library[^2].

The defining characteristic is that validation rules live on the class
definition rather than in a separate schema. This reads cleanly and keeps the
rule next to the field, but it ties you to two costs: the decorators require
TypeScript's legacy `experimentalDecorators` plus `emitDecoratorMetadata`, and
because decorators only run on real class instances, a plain JSON payload from
an HTTP request must first be turned into a class instance (typically via
class-transformer's `plainToInstance`) before `validate` sees the right
prototype and metadata. The library and its companion class-transformer are
almost always used as a pair for exactly this reason.

The project is widely depended upon (tens of millions of npm downloads per
week via the NestJS ecosystem) but maintenance cadence is slow: the repo
carries a few hundred open issues and commits land in bursts rather than
continuously[^3]. It is mature and stable rather than actively evolving, and
its schema-first competitors (notably Zod) have absorbed much of the mindshare
for greenfield TypeScript projects.

## Getting Started

```sh
npm install class-validator class-transformer reflect-metadata
```

```jsonc
// tsconfig.json — decorators + metadata emission are required
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

```typescript
import 'reflect-metadata';
import { validate, IsEmail, Length, Min, Max, IsInt } from 'class-validator';
import { plainToInstance } from 'class-transformer';

class CreateUserDto {
  @Length(2, 20)
  name: string;

  @IsEmail()
  email: string;

  @IsInt()
  @Min(0)
  @Max(120)
  age: number;
}

// A raw request body is a plain object — convert it first so decorators apply.
const dto = plainToInstance(CreateUserDto, { name: 'A', email: 'nope', age: 200 });

const errors = await validate(dto);
if (errors.length) {
  // errors[i].property, errors[i].constraints => { isEmail: '...', min: '...' }
  console.log(errors);
}
```

## Architecture / How It Works

Each decorator (`@IsEmail`, `@Min`, `@ValidateNested`, ...) does not validate
at decoration time; it registers metadata into a global singleton
`MetadataStorage` keyed by the target class and property. Calling
`validate(instance)` walks that storage, resolves the rules for the instance's
constructor (and its ancestors — decorators are inherited across `extends`),
and runs each constraint's `validate` function against the property value[^4].

Constraints come in two flavors. Built-in string/number checks
(`@IsEmail`, `@IsUrl`, `@IsUUID`, `@Matches`, ...) are thin wrappers that call
the corresponding function in validator.js. Structural checks
(`@ValidateNested`, `@IsArray`, `each: true`, `@ValidatePromise`) are handled
by class-validator itself and drive recursion into nested class instances,
array/set/map elements, and resolved promises.

Validation is asynchronous by default: `validate` returns a `Promise`, because
a custom `ValidatorConstraintInterface` may perform async work (a DB uniqueness
check, for example). A synchronous `validateSync` exists but will silently skip
any async constraints. Errors are returned as a tree of `ValidationError`
objects — `{ property, value, constraints, children }` — where `children`
carries nested-object failures, so consumers serializing errors over HTTP must
walk the tree themselves.

Two design consequences follow from the global `MetadataStorage`. First,
metadata is process-global, so duplicate copies of the package in a single
`node_modules` tree register into different storages and validation silently
does nothing — hence the README's insistence on a flattened dependency tree
(npm 6+)[^5]. Second, because decorators run against runtime prototypes, the
library depends on TypeScript's *legacy* decorator implementation and the
`design:type` metadata that `emitDecoratorMetadata` emits; it does not support
the TC39 standard (stage-3) decorators that TypeScript 5 enables without
`experimentalDecorators`.

## Production Notes

- **Plain objects don't validate.** Forgetting `plainToInstance` (or NestJS's
  `transform: true`) is the most common footgun: `validate({...})` on a bare
  object runs zero decorators and returns an empty error array, reading as a
  pass. Always validate a real class instance.
- **`forbidUnknownValues` and the bypass CVE.** Older versions defaulted to
  letting unknown/non-object values through validation, a documented bypass.
  Current versions default `forbidUnknownValues: true`, and the README
  explicitly warns against turning it off — doing so can let unknown objects
  pass validation entirely[^6].
- **Duplicate installs = silent no-op.** Because rule metadata lives in a module
  singleton, two versions of class-validator in the tree (common with monorepos
  / peer-dep mismatches) split the storage and validation quietly stops working.
- **Standard decorators are unsupported.** Projects on TypeScript 5 standard
  decorators, or toolchains that don't emit `design:type` metadata (some
  esbuild/SWC/Vite setups), need explicit legacy-decorator configuration or the
  metadata simply isn't there. This blocks some modern build pipelines.
- **Error shape is verbose.** The nested `ValidationError` tree is not an
  HTTP-ready payload; most teams write (or lean on NestJS's) flattening logic,
  and set `validationError: { target: false }` to avoid echoing the whole input
  object back in responses.
- **Async everywhere.** Even all-synchronous rule sets return a promise; mixing
  in `validateSync` to avoid the await will skip async constraints without
  warning.
- **Maintenance latency.** Bug reports and PRs can sit for extended periods.
  Treat the library as stable-and-slow; do not expect rapid fixes for edge
  cases.

## When to Use / When Not

**Use when:**
- You are on NestJS — it is the idiomatic, first-class validation layer.
- Your team already uses decorators (class-transformer, TypeORM entities) and
  wants validation co-located on the same classes.
- You need instance-shaped domain objects, not just parsed data, after
  validation.

**Avoid when:**
- You want the validated type inferred from the schema (Zod/Yup infer a static
  type; class-validator does not derive types from rules).
- Your build can't emit legacy decorator metadata, or you've moved to TC39
  standard decorators.
- You're validating plain data at an API boundary and don't want the
  class-instance + `plainToInstance` round-trip.

## Alternatives

- colinhacks/zod — schema-first validation with static type inference; the
  default choice for new TypeScript projects that don't need decorators.
- jquense/yup — object-schema validation popular in React/Formik forms; use
  when you want a chainable schema rather than class annotations.
- ajv-validator/ajv — JSON Schema validation, very fast; use when your contract
  is already JSON Schema or you validate at high throughput.
- validatorjs/validator.js — the low-level string validators class-validator
  wraps; use directly when you only need `isEmail`-style checks, no schema.
- sinclairzx81/typebox — build JSON Schema and derive TypeScript types from one
  definition; use when you want schema + types + JSON Schema output together.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015-10 | Initial release under the typestack org; decorator API built on validator.js.[^3] |
| 0.13.0 | 2021 | Broad decorator additions and validator.js updates. |
| 0.14.0 | 2022 | `forbidUnknownValues` defaulted to `true`, closing the unknown-value validation bypass.[^6] |
| 0.14.1 | 2024 | Maintenance and dependency updates on the 0.14 line. |

(class-validator has never shipped a 1.0; it has lived on the 0.x line for its
entire history despite heavy production use.)

## References

[^1]: NestJS validation techniques (built on class-validator + class-transformer). https://docs.nestjs.com/techniques/validation
[^2]: validator.js — the underlying string validation library. https://github.com/validatorjs/validator.js
[^3]: typestack/class-validator repository and releases. https://github.com/typestack/class-validator/releases
[^4]: class-validator README — usage, custom constraints, nested validation. https://github.com/typestack/class-validator#readme
[^5]: class-validator README — npm 6+ flattened-dependency-tree requirement. https://github.com/typestack/class-validator#installation
[^6]: class-validator README — `forbidUnknownValues` default and security warning. https://github.com/typestack/class-validator#passing-options

## Tags

typescript, javascript, validation, decorators, nestjs, class-transformer, dto, runtime-validation, backend, node
