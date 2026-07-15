# thephpleague/fractal

> A presentation and transformation layer for API output in PHP — a view layer for your JSON.

[GitHub repo](https://github.com/thephpleague/fractal) ·
[Official website](https://fractal.thephpleague.com) ·
[License: MIT](https://github.com/thephpleague/fractal/blob/master/LICENSE)

Composer package: `league/fractal` (the Packagist name; the GitHub repo lives under
the `thephpleague` organization).

## Overview

Fractal is a framework-agnostic PHP library that sits between your source data
(usually database models) and your serialized API output. Its thesis is that
passing an ORM record straight to `json_encode()` leaks your schema to clients:
column renames, type drift, and accidental field exposure all become breaking
changes. Fractal inserts an explicit transformer layer so the output shape is
declared in code, decoupled from storage[^1].

The library was created in 2013 by the PHP League and popularized in the Laravel
and API-building community, in part through Phil Sturgeon's writing on
pragmatic REST[^2]. Its three defining primitives are **transformers** (per-type
mapping classes), **serializers** (envelope strategies like plain arrays,
data-wrapped arrays, or JSON:API), and **includes** (opt-in embedding of related
resources requested by the client). It ships pagination adapters and a cursor
abstraction for large sets.

The defining tension is verbosity versus control. Fractal makes you write one
transformer class per resource type and wire includes by hand, which is more
boilerplate than reflection-based serializers — but it gives precise,
reviewable control over every field and every relationship. The second tension
is ecosystem: since Laravel shipped Eloquent API Resources in 2017, much of
Fractal's Laravel mindshare moved to the framework-native option, leaving Fractal
strongest where you need framework independence, a real JSON:API serializer, or
client-driven include parsing.

## Getting Started

```bash
composer require league/fractal
```

```php
use League\Fractal\Manager;
use League\Fractal\Resource\Collection;
use League\Fractal\TransformerAbstract;

class BookTransformer extends TransformerAbstract
{
    protected array $availableIncludes = ['author'];

    public function transform(Book $book): array
    {
        return [
            'id'    => (int) $book->id,   // explicit casts — no schema leaks
            'title' => $book->title,
            'year'  => (int) $book->yr,
        ];
    }

    public function includeAuthor(Book $book)
    {
        return $this->item($book->author, new AuthorTransformer());
    }
}

$fractal  = new Manager();
$fractal->parseIncludes('author');       // usually from ?include=author

$resource = new Collection($books, new BookTransformer(), 'books');
$payload  = $fractal->createData($resource)->toArray();
```

Swap the envelope by calling `$fractal->setSerializer(new JsonApiSerializer())`
before `createData`.

## Architecture / How It Works

`Manager` is the entry point. It holds the active serializer and the list of
requested includes, and it turns a **resource** into a **Scope**. Resources come
in a few shapes: `Item` (one record), `Collection` (a list, optionally with a
`PaginatorInterface` or `CursorInterface`), plus `NullResource` and primitive
resources. Each carries the data, a transformer, and an optional resource key.

`TransformerAbstract` defines `transform()` for the base fields and any number of
`includeXxx()` methods. Includes are declared via `availableIncludes` (opt-in,
requested by the client) and `defaultIncludes` (always embedded). An
`includeXxx()` method returns another resource (`$this->item(...)` /
`$this->collection(...)`), so transformation is recursive: Fractal walks the
include tree, building a child Scope for each level and matching it against the
dotted include paths the client asked for (e.g. `author.publisher`).

`Scope` is the node in that recursion tree. It knows its position in the include
path, decides whether a requested include applies at the current depth, and
merges child output back into the parent. A recursion limit (default 10) guards
against runaway nesting.

**Serializers** control only the envelope, not the field mapping. `ArraySerializer`
emits flat arrays; `DataArraySerializer` (the default) wraps payloads in a `data`
key and metadata in `meta`; `JsonApiSerializer` produces spec-shaped
`type`/`id`/`attributes`/`relationships`/`included` documents, inferring `type`
from the resource key. Because the serializer is orthogonal to the transformer,
the same transformers produce different wire formats without change.

The library depends on no framework. Pagination is bridged through small,
separately-versioned adapter packages (Illuminate, Doctrine, Pagerfanta, Phalcon,
Zend), each living in its own repo under the same org, so Fractal core stays
dependency-light.

## Production Notes

**"0.x" is not a warning.** After more than a decade Fractal has never tagged a
1.0 — the current release is 0.21 (December 2025)[^3]. In practice the public API
has been stable for years; breaking changes are rare, batched into minor bumps,
and documented. Read the version as maintenance-mode maturity, not instability.
Development is slow and mostly limited to PHP-version support and bug fixes.

**Includes cause N+1 queries.** `includeAuthor()` reads `$book->author` per item;
if you did not eager-load the relation, you get one query per row. Fractal does
not touch your database, so you must inspect the requested includes
(`$manager->getRequestedIncludes()`) and eager-load accordingly before building
the resource. This is the single most common performance surprise.

**Deep includes multiply work.** Every level of `parseIncludes` builds a fresh
Scope and re-transforms nested data. Client-controlled `?include=` on public
endpoints should be allow-listed and depth-bounded — leaving `availableIncludes`
open lets callers request expensive nested trees. The default recursion limit is
10 but memory and query cost grow well before that.

**Serializer output shape is a contract.** Switching from `DataArraySerializer`
to `ArraySerializer` (or to JSON:API) changes the envelope your clients parse.
Pick one early. `JsonApiSerializer` is stricter than the plain serializers and
depends on correct resource keys for its `type` field; a missing or wrong key
produces subtly malformed documents. The long-promised HAL serializer in the
README's TODO was never delivered — do not plan around it.

**PHP version floor moves in minors.** Recent releases raised the minimum PHP;
0.21 requires PHP >= 7.4[^3]. Pin the constraint and check the requirement before
upgrading in older codebases.

**Laravel overlap.** On Laravel, Eloquent API Resources cover the common case
natively. Reach for Fractal when you need framework independence, real JSON:API
output, or structured include parsing that Resources handle less directly. The
`spatie/laravel-fractal` wrapper smooths the ergonomics if you stay with Fractal
on Laravel.

## When to Use / When Not

**Use when:**
- You want an explicit, reviewable transformation layer decoupled from your ORM.
- You need framework-agnostic API output (Slim, Symfony, Phalcon, no framework).
- You want JSON:API or a custom serializer with client-driven `?include=` embedding.
- You value hand-written field control over reflection-based auto-serialization.

**Avoid when:**
- You are on Laravel and Eloquent API Resources already meet your needs.
- You want declarative, attribute/annotation-driven mapping with minimal classes.
- You need a full hypermedia API framework (routing, docs, negotiation) out of the box.
- Per-type transformer boilerplate is unacceptable for a very large schema.

## Alternatives

- laravel/framework — Eloquent API Resources are built in; use on Laravel when you don't need framework independence or a formal JSON:API serializer.
- schmittjoh/serializer — JMS Serializer, attribute/annotation-driven mapping; use when you prefer declarative metadata over imperative transformer classes.
- api-platform/core — full hypermedia API framework on Symfony; use when you want routing, negotiation, and OpenAPI docs, not just a transformation layer.
- symfony/serializer — Symfony's native serializer with groups and normalizers; use when you are on Symfony and want the framework's own component.
- spatie/laravel-fractal — thin Laravel wrapper around Fractal itself; use for nicer ergonomics when you have already chosen Fractal on Laravel.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-11-26 | Repository created under the PHP League[^1]. |
| 0.14.0 | 2016-07-21 | Mid-life release during peak Laravel-community adoption. |
| 0.17.0 | 2017-06-12 | Predates/overlaps Laravel's own Eloquent API Resources. |
| 0.18.0 | 2019-05-10 | PHP version floor raised; maintenance cadence. |
| 0.19 | 2020-01-24 | Continued PHP support updates. |
| 0.20 | 2022-03-07 | Modern PHP support. |
| 0.21 | 2025-12-16 | Current release; requires PHP >= 7.4[^3]. |

## References

[^1]: Fractal README and repository, The PHP League. https://github.com/thephpleague/fractal
[^2]: Fractal documentation, "A presentation and transformation layer for complex data output." https://fractal.thephpleague.com
[^3]: Fractal release 0.21, published 2025-12-16; README "Requirements: >= PHP 7.4." https://github.com/thephpleague/fractal/releases

## Tags

php, api, serialization, json-api, rest, data-transformation, laravel, thephpleague, transformer, presentation-layer
