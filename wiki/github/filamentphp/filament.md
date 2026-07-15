# filamentphp/filament

> A Laravel/Livewire framework for building admin panels and CRUD apps in PHP, without writing a frontend.

[GitHub repo](https://github.com/filamentphp/filament) ·
[Official website](https://filamentphp.com) ·
[License: MIT](https://github.com/filamentphp/filament/blob/4.x/LICENSE.md)

## Overview

Filament is a collection of Laravel packages for assembling data-driven back-office interfaces — admin panels, internal tools, dashboards — by declaring them in PHP rather than building a separate frontend[^1]. You describe a form, a table, or a detail view as an array of component objects in a PHP class, and Filament renders a server-driven, reactive UI. It is built on the "TALL stack": Tailwind CSS, Alpine.js, Laravel, and Livewire[^2].

The defining tradeoff follows directly from Livewire[^3]: there is no client-side application and no API contract. Component state lives on the server, is serialized into the HTML, and every reactive interaction (a dependent select, a live-validated field, a table filter) issues an AJAX round-trip that re-renders the affected component. This is what lets a PHP developer ship a polished CRUD panel in an afternoon with zero JavaScript — and it is also the ceiling. Latency is bounded by a network hop, payload grows with component state, and none of the UI is reusable outside a Blade/Livewire page.

Filament targets the large population of Laravel teams who need internal tooling and want to stay in one language and one mental model. It is not a public-facing site framework, not headless, and not API-first. Within its niche — the admin surface of a Laravel monolith — it is one of the most complete options available, and the momentum around it (a paid ecosystem of plugins and themes has formed) reflects that.

## Getting Started

```bash
composer require filament/filament
php artisan filament:install --panels
php artisan make:filament-user
```

A resource ties an Eloquent model to auto-generated CRUD pages:

```php
// app/Filament/Resources/PostResource.php
use Filament\Forms;
use Filament\Tables;
use Filament\Resources\Resource;

class PostResource extends Resource
{
    protected static ?string $model = Post::class;

    public static function form(Forms\Form $form): Forms\Form
    {
        return $form->schema([
            Forms\Components\TextInput::make('title')->required()->maxLength(255),
            Forms\Components\Select::make('author_id')
                ->relationship('author', 'name')->searchable(),
            Forms\Components\RichEditor::make('body'),
            Forms\Components\Toggle::make('published'),
        ]);
    }

    public static function table(Tables\Table $table): Tables\Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('title')->searchable()->sortable(),
                Tables\Columns\IconColumn::make('published')->boolean(),
            ])
            ->filters([Tables\Filters\TrashedFilter::make()]);
    }
}
```

The `List`, `Create`, `Edit`, and `View` pages are generated from these two methods.

## Architecture / How It Works

Filament is not a single package but a set of composable builders, historically published separately and now consolidated under one repo and version line:

- **Panel Builder** — the admin surface. A panel registers resources, pages, and widgets; multiple panels (e.g. `admin`, `app`) can coexist with independent auth, routing, and theming.
- **Form Builder / Schema** — the `Field` component tree. Fields are stateful, support reactivity (`live()`, `afterStateUpdated()`), conditional visibility, and relationship binding.
- **Table Builder** — columns, filters, actions, and bulk actions over an Eloquent query. Pagination, sorting, and search are server-side.
- **Infolists** — read-only structured record views (the view counterpart to forms).
- **Actions** — reusable, modal-driven operations (confirm, fill a form, mutate, notify).
- **Notifications** and **Widgets** — in-app flash messages and dashboard cards/charts.

Under the hood, each page is a **Livewire component**. Alpine.js handles purely client-side behavior (dropdowns, modals, keyboard nav); Livewire handles anything that touches server state. When you type in a `live()` field, Livewire serializes the component's public properties, POSTs them, runs the PHP lifecycle, diffs the rendered Blade output, and patches the DOM. The "schema" you declare is walked on every render to reconstruct the component tree — Filament is doing meaningful work per request, not caching a compiled view.

Styling is Tailwind with a Filament preset. Custom themes require compiling CSS through the project's Vite/Tailwind build against Filament's config[^4]; you cannot restyle deeply with runtime configuration alone. This couples your panel's appearance to a working asset-build pipeline.

## Production Notes

**The round-trip is the performance model.** Every reactive interaction is a network request plus a full Livewire lifecycle. On a fast connection this is invisible; on high-latency links or with very large forms it is felt. Reserve `live()` / `reactive()` for fields that genuinely need it — making a whole form live turns each keystroke (debounced) into a server round-trip.

**N+1 queries are the default failure mode.** Table columns that read relationships (`author.name`, counts, aggregates) will issue a query per row unless you eager-load. Filament exposes `modifyQueryUsing()` / `->with()` hooks, but the framework will not add eager loads for you. Profiling tables under realistic row counts is mandatory before shipping.

**State is serialized to the browser.** Livewire embeds component public properties in the page and in every payload. Loading a full Eloquent model into a form means its attributes cross the wire; sensitive columns can leak if you bind a whole record without narrowing. Large collections held in component state also inflate payload size on every request.

**Authorization is opt-in.** Filament integrates with Laravel policies, but a resource with no policy is open to any panel user. Access control (per-resource, per-action, per-field) must be wired explicitly — there is no deny-by-default. This is the most common security footgun in Filament deployments.

**Upgrades are real migrations.** The v2 → v3 jump consolidated the previously separate form/table packages into the panel line and moved namespaces, requiring code changes across every resource[^5]. Treat major-version upgrades as scheduled work, not a `composer update`. Version compatibility is pinned tightly to Laravel, Livewire (v3), and PHP (8.2+) ranges.

**Scale caveats.** Global search across many resources, wide tables with many live filters, and dashboards with several querying widgets all multiply per-request DB work. Filament suits internal tools with tens to low-hundreds of concurrent operators well; it is not designed as a high-QPS public application layer.

## When to Use / When Not

**Use when:**
- You have a Laravel app and need an admin panel, internal tool, or CRUD back-office fast.
- Your team is PHP-first and wants to avoid a separate JS frontend and API layer.
- The interface is behind auth and used by staff, not the public internet at scale.
- You want form/table/action primitives you can extend in PHP rather than a rigid scaffold.

**Avoid when:**
- You are not on Laravel — Filament is not portable to other backends.
- You need an API-first or headless setup, a mobile client, or a SPA that reuses the UI.
- The surface is a high-traffic public frontend where per-interaction round-trips and SEO matter.
- You need rich, low-latency client interactivity (canvas, real-time editors) that Livewire's model fights.

## Alternatives

- laravel/nova — official first-party Laravel admin panel; paid license, more turnkey, less open to deep customization. Use when you want vendor support and don't mind paying.
- refinedev/refine — React, API-first admin framework. Use when your backend is headless and your team wants a JS/TypeScript frontend.
- directus/directus — headless data platform with an admin UI over any SQL database. Use when you want a database-first admin decoupled from a specific app framework.
- strapi/strapi — Node headless CMS with an admin panel. Use when the primary need is content modeling with a REST/GraphQL API.
- z-song/laravel-admin — older Laravel admin scaffolder. Use only for legacy projects already committed to it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-06 | Initial admin-panel release on the TALL stack[^1]. |
| 2.0 | 2021-11 | Form and table builders usable standalone; broader component set. |
| 3.0 | 2023-08 | Consolidation into the Panel Builder; packages unified, Livewire v3[^5]. |
| 4.x | 2025 | Current major line; Laravel 11+, Livewire v3, PHP 8.2+ (per repo badges). |

## References

[^1]: Filament documentation — introduction and overview. https://filamentphp.com/docs
[^2]: Repository topics and README: TALL stack (Tailwind, Alpine.js, Laravel, Livewire). https://github.com/filamentphp/filament
[^3]: Livewire — server-rendered, reactive components for Laravel. https://livewire.laravel.com
[^4]: Filament documentation — theming / custom themes via Tailwind + Vite. https://filamentphp.com/docs
[^5]: Filament v3 upgrade guide (package consolidation, namespace changes). https://filamentphp.com/docs/3.x/upgrade-guide

## Tags

php, laravel, livewire, admin-panel, crud, tall-stack, tailwind-css, alpine-js, forms, tables, low-code, backend
