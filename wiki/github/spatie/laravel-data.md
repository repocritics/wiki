# spatie/laravel-data

> Typed data objects for Laravel that collapse form requests, API resources, and DTOs into one class definition.

[GitHub repo](https://github.com/spatie/laravel-data) ·
[Official website](https://spatie.be/docs/laravel-data/) ·
[License: MIT](https://github.com/spatie/laravel-data/blob/main/LICENSE.md)

## Overview

`laravel-data` is a Spatie package, first released in 2021[^1], that lets you
describe a piece of data once as a PHP class and reuse that single definition
as a request validator, an API transformer (Laravel Resource replacement), a
data transfer object, and a source of generated TypeScript types. A "data
object" is a plain PHP class extending `Spatie\LaravelData\Data`, typically
with a promoted-property constructor; the package reads those typed properties
via reflection and derives validation rules, casting, and serialization from
the type system and attributes.

The defining tension is DRY-versus-magic. The pitch — write your shape once
instead of three times (request, resource, DTO) — is genuinely useful on
codebases where those three drift apart. The cost is that a lot of behavior is
inferred at runtime from reflection and attributes, so the mapping between "my
class" and "the validation rules / JSON that actually come out" is not always
obvious, and the package carries real per-request cost unless you cache its
structure resolution. It is best understood as a framework-tier abstraction,
not a lightweight value-object library.

The package is maintained as part of Spatie's open-source portfolio and is
actively developed; the current major line is v4, with the codebase seeing
regular commits into 2026[^2]. Primary author is Ruben Van Assche.

## Getting Started

```bash
composer require spatie/laravel-data
```

```php
use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Max;

class SongData extends Data
{
    public function __construct(
        public string $title,
        #[Max(255)]
        public string $artist,
    ) {
    }
}

// From a request — validation runs automatically from the types + attributes
$song = SongData::from($request);

// From an array / model / anything
$song = SongData::from(['title' => 'Rendez-Vous', 'artist' => 'Jarre']);

// Back out as a resource (respects lazy/only/except)
return SongData::from($song)->toResponse($request);
```

A controller can typehint `SongData` directly and Laravel resolves it from the
request, validating in the process — replacing a `FormRequest`.

## Architecture / How It Works

The core is a reflection-driven pipeline. When you call `SongData::from(...)`,
the package resolves a `DataClass` structure — the list of properties, their
types, attributes, casts, and mappers — then runs a pipeline of stages
(authorize, map property names, validate, cast values, fill the object). Output
follows the reverse path: a `DataObject` is walked, each property transformed
(via type-specific `Transformer`s), and lazy properties are included or omitted
based on the request.

Key concepts layered on top of plain properties:

- **Casts and Transformers** — casts turn incoming scalars into rich types
  (e.g. `Carbon`, enums, nested `Data` objects); transformers do the reverse on
  output. Both are resolvable globally in config or per-property via attributes.
- **Lazy properties** — `Lazy::create(fn () => ...)` defers computation and
  lets the caller opt in with `->include('field')`, mirroring how API Resources
  conditionally load relations.
- **Validation attributes** — `#[Required]`, `#[Max]`, `#[Rule]`, etc. compile
  to Laravel validation rules; many rules are also inferred from the PHP type
  (a non-nullable `string` implies `required|string`).
- **Name mappers** — `#[MapInputName]` / `#[MapOutputName]` bridge snake_case
  JSON and camelCase PHP without hand-written key maps.
- **TypeScript generation** — via the sibling `spatie/typescript-transformer`,
  data classes emit `.d.ts` types so the frontend shares the backend's shape.

Everything hangs off runtime reflection, so the package co-evolves closely with
Laravel's container and validation internals. That coupling is total: this is
not a portable DTO library, it is a Laravel subsystem.

## Production Notes

- **Structure caching is not optional at scale.** Resolving a data class by
  reflection on every request is measurable overhead. The package ships
  `php artisan data:cache-structures` to precompute and cache the resolved
  class structures; run it in your deploy pipeline (like `config:cache`).
  Without it, high-throughput endpoints pay reflection cost per request[^3].
- **Major upgrades have been substantial.** v2, v3, and v4 each carried
  breaking changes — casts/transformers signatures, validation resolution, and
  attribute namespaces have moved between majors. Budget real time for major
  bumps and read the upgrade guide rather than treating it as a patch[^4].
- **Validation inference can surprise.** Because rules come partly from PHP
  types and partly from attributes, the effective ruleset for a class is not
  visible in one place. Nullable-vs-optional, `present`, and nested-object
  validation are the usual sources of "why did (or didn't) this fail" tickets.
- **Nested and collection data multiplies cost.** Deeply nested data objects
  and large `DataCollection`s allocate and transform per element; profile
  before returning thousands of items through the transformation pipeline.
- **PHP/Laravel version floor moves fast.** v4 targets recent PHP (8.1+) and
  current Laravel majors; the package does not carry long back-compat tails, so
  it effectively pins you to a fairly current stack.
- **It is an abstraction you commit to.** Once controllers, requests, and
  resources are expressed as data objects, backing out is a rewrite. Adopt it
  as an architectural decision, not a per-endpoint convenience.

## When to Use / When Not

**Use when:**
- You maintain the same shape in three places (FormRequest, API Resource, DTO)
  and they drift apart.
- You want typed request/response contracts and free TypeScript types shared
  with a frontend.
- You are on a current Laravel + PHP version and comfortable with reflection-
  heavy, attribute-driven code.

**Avoid when:**
- You need a framework-agnostic or dependency-light value object — use a plain
  DTO or `cuyz/valinor` instead.
- Your app is a handful of endpoints where a FormRequest plus an array is
  simpler than learning the package's casting/lazy/mapper model.
- You cannot run the structure-cache step in deploy and have latency-sensitive,
  high-QPS endpoints.

## Alternatives

- spatie/data-transfer-object — Spatie's older, simpler DTO package; effectively
  superseded by laravel-data but lighter if you only need plain typed objects.
- cuyz/valinor — framework-agnostic strict object mapping from raw input; use
  when you want typed hydration without Laravel coupling.
- spatie/laravel-typescript-transformer — use directly when you only want
  TS type generation and not the full data-object stack.
- laravel FormRequest + API Resources (built-in) — use when the three-places
  duplication is small and you would rather not add an abstraction.
- thephpleague/fractal — transformer-only layer for API output when you do not
  need validation or DTO hydration unified.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2021 | Initial release; data objects for requests, resources, DTOs[^1]. |
| v2.0 | 2022 | Major rework of casts/transformers and validation resolution. |
| v3.0 | 2023 | Further breaking changes; expanded attribute and mapping system. |
| v4.x | 2024–2026 | Current line; recent PHP/Laravel floor, ongoing development[^2]. |

(Exact release dates per major are on Packagist / GitHub Releases; only the
first release year is asserted here with confidence.)

## References

[^1]: Spatie blog / package announcement for laravel-data (2021). https://spatie.be/docs/laravel-data
[^2]: GitHub repository — commit and release activity. https://github.com/spatie/laravel-data
[^3]: Optimizing data objects — structure caching (`data:cache-structures`). https://spatie.be/docs/laravel-data/v4/advanced-usage/optimizing
[^4]: Upgrade guide. https://spatie.be/docs/laravel-data/v4/upgrading

## Tags

php, laravel, data-transfer-object, validation, api-resource, typescript, serialization, dto, spatie, backend
