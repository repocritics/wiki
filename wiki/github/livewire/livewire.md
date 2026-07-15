# livewire/livewire

> A full-stack framework for Laravel that builds dynamic, reactive UIs from server-rendered PHP — no separate frontend app.

[GitHub repo](https://github.com/livewire/livewire) ·
[Official website](https://livewire.laravel.com) ·
[License: MIT](https://github.com/livewire/livewire/blob/main/LICENSE.md)

## Overview

Livewire lets you build interactive interfaces in Laravel without writing a JavaScript SPA. You author a component as a PHP class plus a Blade template; user interactions trigger AJAX round-trips to the server, which re-renders the component and ships back an HTML diff that Livewire morphs into the live DOM[^1]. The programming model is "server-rendered but reactive" — the mental model of a classic MVC app with the interactivity of a client framework, at the cost of a network hop per interaction.

The project was created by Caleb Porzio and first released in 2019. It is closely tied to Alpine.js (also Porzio's), which handles the client-side sprinkles Livewire deliberately does not: version 3 bundles Alpine and exposes two-way bridges (`@entangle`, `wire:model`) between server state and client state[^2]. Livewire is not a general-purpose framework — it only runs inside Laravel, and it assumes Blade.

The defining tension is latency versus simplicity. Livewire removes the need for a REST/GraphQL API, a JS build pipeline, and client-side state management — you stay in PHP. In exchange, every stateful interaction that isn't handled purely by Alpine costs a request to the server, and the full component state travels over the wire on each round-trip. For CRUD dashboards and forms this is an excellent trade; for high-frequency interactive UIs it is the wrong tool.

## Getting Started

```bash
composer require livewire/livewire
php artisan make:livewire counter
```

```php
// app/Livewire/Counter.php
namespace App\Livewire;

use Livewire\Component;

class Counter extends Component
{
    public int $count = 0;         // public props = component state, sent to client

    public function increment(): void
    {
        $this->count++;
    }

    public function render()
    {
        return view('livewire.counter');
    }
}
```

```blade
{{-- resources/views/livewire/counter.blade.php --}}
<div>
    <button wire:click="increment">+</button>
    <span>{{ $count }}</span>
</div>
```

Drop it into any Blade view with `<livewire:counter />` (or route directly to a full-page component). Each `wire:click` issues a request; the server re-runs `render()` and returns the patched HTML.

## Architecture / How It Works

A Livewire component has a server lifecycle built on **snapshots**. On first render the component is *dehydrated* into a JSON snapshot of its public properties plus a checksum, embedded in the page. On each interaction the browser POSTs that snapshot back; Livewire *hydrates* a fresh instance from it, applies the queued action (method call or property update), re-renders the Blade view, diffs the new HTML against the old, and returns the changes[^1]. The client library morphs the DOM in place rather than replacing it, preserving focus, scroll, and Alpine state.

Key mechanics:

- **State lives on the wire.** Only public properties are serialized. They must be JSON-serializable or a supported type (Eloquent models, collections, enums, `DateTime`, form objects) that Livewire knows how to hydrate. The snapshot is signed with a checksum so a tampered payload is rejected — but the values themselves are visible and editable in the browser, so they are untrusted input.
- **Directives.** `wire:model`, `wire:click`, `wire:submit`, `wire:poll`, `wire:navigate`, `wire:loading`, `wire:dirty`, keyed `wire:key`. In v3, `wire:model` is deferred by default (syncs on the next action); `.live` opts into per-keystroke syncing.
- **Alpine bridge.** `@entangle` and `wire:model` share reactive state between Livewire (server) and Alpine (client), so you can keep a modal's open/close purely client-side while its form submits through Livewire.
- **PHP attributes (v3).** `#[Computed]` for memoized derived state, `#[Validate]` for inline rules, `#[On('event')]` for listeners, `#[Locked]` to forbid client-side mutation of a property, `#[Url]` to bind a property to the query string.
- **`wire:navigate`** turns full-page component links into SPA-style swaps (fetch + morph the `<body>`), giving persistent layout and no full reload without an actual SPA.

Two higher-level surfaces sit on top: **Volt**, a functional single-file component API, and **Flux**, the official (partly paid) UI component library. Both are separate packages, not in this repo.

## Production Notes

- **Every interaction is a server request.** This is the central operational fact. A form with `wire:model.live` on ten fields can generate a request per keystroke. Default to deferred `wire:model`, reach for `.live` only where you need it, and push transient UI state (toggles, dropdowns, tabs) into Alpine to avoid the round-trip entirely.
- **Payload size grows with state.** The entire public-property snapshot is sent on every request. Large collections, hydrated Eloquent models, or big arrays held as public props inflate every round-trip. Keep heavy data in `#[Computed]` methods (recomputed server-side, never serialized) rather than public properties.
- **Public properties are client-controlled input.** Anything the client can see it can also modify before sending it back. Never trust a public property for authorization; use `#[Locked]` on IDs and always re-authorize server-side. This is the most common Livewire security footgun.
- **File uploads and long requests.** Uploads use a temporary-file dance (`WithFileUploads`); large uploads and slow actions block the component's request queue, so show `wire:loading` states and consider queued jobs for heavy work.
- **The v2 → v3 upgrade was a rewrite.** v3 changed the namespace from `App\Http\Livewire` to `App\Livewire`, replaced the JS core, bundled Alpine (removing the need to include it separately, and causing double-Alpine bugs for apps that still did), and swapped several APIs (`emit` → `dispatch`, lifecycle hook signatures). There is an official upgrade guide and a `livewire:upgrade` command, but it is not a drop-in bump[^2].
- **Debugging is split across two runtimes.** A broken interaction can be a server exception, a hydration/serialization error, a checksum mismatch, or an Alpine/morph issue on the client. Livewire surfaces server errors in the browser console and network tab; get comfortable reading the request/response payloads.
- **Testing.** Livewire ships a first-class server-side test API (`Livewire::test(Counter::class)->call('increment')->assertSee(...)`), which covers most logic without a browser. Genuinely client-side behavior still needs Dusk/Playwright.

## When to Use / When Not

**Use when:**
- You're already on Laravel and want interactivity without standing up a separate frontend, API layer, or JS build.
- The UI is CRUD-shaped: dashboards, admin panels, forms, tables, wizards, filtered lists.
- Your team is PHP-strong and JS-light, and you value staying in one language and one mental model.
- You want server-rendered HTML (SEO, no hydration-of-a-JSON-blob) with sprinkles of reactivity.

**Avoid when:**
- The interface is latency-sensitive or high-frequency (drawing canvases, real-time collaborative editors, drag-heavy interactions) — the per-interaction round-trip will hurt.
- You're offline-first or need to keep working without a server connection.
- Your frontend is genuinely complex and app-like; a real SPA (Inertia + Vue/React, or a standalone framework) will scale better.
- You're not on Laravel. Livewire has no meaning outside it.

## Alternatives

- inertiajs/inertia — use instead when you want real Vue/React components with client-side routing but no separate API; Inertia keeps the SPA model, Livewire keeps the server model.
- hotwired/turbo — use instead in Rails or non-Laravel stacks for the same server-rendered-HTML-over-the-wire philosophy.
- bigskysoftware/htmx — use instead when you want the round-trip-HTML pattern framework-agnostically, without PHP components or a stateful snapshot.
- phoenixframework/phoenix_live_view — use instead when you're on Elixir; the same idea over a persistent WebSocket rather than AJAX.
- alpinejs/alpine — use *alongside* (or instead, for purely client-side widgets) when no server state is involved; Livewire already bundles it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019 | Initial public release by Caleb Porzio. |
| 1.0 | 2020 | First stable line. |
| 2.0 | 2020-09 | API maturation, performance work, broad adoption. |
| 3.0 | 2023-11 | Full rewrite: new JS core, Alpine bundled, `wire:navigate`, PHP-attribute API (`#[Computed]`, `#[Validate]`, `#[Locked]`), namespace change, `emit`→`dispatch`[^2]. |

## References

[^1]: Livewire documentation, "How Livewire Works" / hydration. https://livewire.laravel.com/docs/hydration
[^2]: Livewire documentation, "Upgrading from Livewire 2" and v3 release notes. https://livewire.laravel.com/docs/upgrading

## Tags

php, laravel, full-stack, blade, reactive-ui, server-rendered, frontend, ajax, alpinejs, mit-license
