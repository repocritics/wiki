# sveltejs/svelte

> Cybernetically enhanced web apps — a compile-time UI framework that disappears at runtime.

[GitHub repo](https://github.com/sveltejs/svelte) ·
[Official website](https://svelte.dev) ·
[License: MIT](https://github.com/sveltejs/svelte/blob/main/LICENSE.md)

## Overview

Svelte compiles components into surgical, framework-less JavaScript at build time, rather than shipping a runtime that diffs a virtual DOM in the browser[^1]. The pitch since 2019 has been "the framework that disappears": output is roughly the imperative DOM code you would have written by hand, plus a small reactivity helper. In practice the runtime is not literally zero — there is a `svelte/internal` module — but it is an order of magnitude smaller than React's reconciler.

Svelte 5 (2024-10) replaced the magic-assignment reactivity model of earlier versions with **runes** (`$state`, `$derived`, `$effect`)[^2]. This was the most significant break in the project's history: it traded "looks like vanilla JS" for an explicit signals model closer to Solid or Vue's Composition API. Existing Svelte 3/4 code continues to work in legacy mode, but new code is expected to use runes.

The companion framework `SvelteKit` is the standard application shell — file-based routing, server endpoints, adapter-driven deploy targets. Most production Svelte usage runs through SvelteKit, not bare Svelte components.

## Getting Started

```bash
npx sv create my-app
cd my-app
npm install
npm run dev
```

A minimal Svelte 5 component with runes:

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>

<button onclick={() => count++}>
  count: {count} (doubled: {doubled})
</button>
```

## Architecture / How It Works

The Svelte compiler is a multi-stage pipeline: parse SFC → AST transform → analyze reactivity dependencies → emit JS + CSS. The emitted code in Svelte 5 uses a fine-grained reactivity runtime that tracks signal reads inside effect scopes, similar in spirit to Solid[^3]. This is a meaningful departure from Svelte 3/4, which used compile-time dependency analysis on bare variable assignments.

Two compilation targets coexist:

1. **Client mode** — hydrates on the client, runs effects in response to signal mutations.
2. **Server mode** — emits string-concatenation SSR output, no reactivity at runtime.

Runes are not a runtime API in the JavaScript sense. They look like function calls but are recognized syntactically by the compiler and rewritten into direct signal reads/writes. Calling `$state(0)` outside a `.svelte` or `.svelte.js` file throws. This is the same design tradeoff Vue's macros use.

## Production Notes

**Bundle size.** Real-world SvelteKit apps in 2025 land around 10–30 KB gzipped for an empty route, growing roughly linearly with feature count. The marketing "0kb" headline has not been accurate since Svelte 3; it refers to per-component overhead, not the framework baseline.

**Runes migration.** The Svelte 3/4 → 5 migration is non-trivial despite the legacy mode. Reactive declarations (`$:`), stores, and `bind:` semantics all have subtly different timing under runes. There is a `npx sv migrate svelte-5` codemod, but it leaves behind warnings to resolve manually[^4]. Teams report 1–3 days of migration work per medium application.

**SSR + reactivity edge cases.** `$effect` does not run on the server. Code that mutates state inside an effect to populate the initial render will silently produce empty SSR output. Use `$derived` or move the work into a load function.

**Ecosystem maturity.** The Svelte ecosystem is smaller than React's by 1–2 orders of magnitude. UI kits (skeleton, shadcn-svelte, melt-ui), form libraries (sveltekit-superforms), and animation libraries exist and are production-quality, but niche needs frequently force you to write the integration yourself.

**Stores legacy.** `svelte/store` (`writable`, `readable`, `derived`) is not deprecated but is no longer the recommended state model. New code should prefer `$state` rune classes in `.svelte.ts` files. Mixed codebases work but require care around store subscriptions inside rune scopes.

## When to Use / When Not

**Use when:**
- You want the smallest realistic bundle for an interactive app.
- You prefer single-file components with HTML-first templates.
- You are building a content site or admin tool where SvelteKit's load-function model fits.
- You want first-class transitions and animations without a separate library.

**Avoid when:**
- You need a mature mobile story — there is no Svelte Native equivalent of React Native in serious production use.
- Your team is large and you cannot afford the smaller hiring pool relative to React.
- You depend on third-party React component libraries with no Svelte equivalent.
- You need to share a UI codebase across web and native.

## Alternatives

- [facebook/react](../facebook/react.md) — larger ecosystem, native story, but heavier runtime.
- [vuejs/core](../vuejs/core.md) — similar SFC ergonomics, Proxy-based reactivity, larger ecosystem.
- [solidjs/solid](../solidjs/solid.md) — same fine-grained reactivity philosophy with JSX instead of SFCs.
- [withastro/astro](../withastro/astro.md) — can host Svelte islands without committing the whole app to Svelte.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-11 | Initial Rich Harris release, "the magical disappearing framework"[^1]. |
| 3.0 | 2019-04 | Reactive assignments, SFC syntax overhaul[^5]. |
| 4.0 | 2023-06 | Performance pass, no API breaks. |
| 5.0 | 2024-10 | Runes, signals-based reactivity, snippets replace slots[^2]. |
| 5.x | 2025 | SvelteKit 2.x cycle, asset routing, remote functions preview. |

## References

[^1]: Rich Harris, "Frameworks Without the Framework" — 2016-12-27. https://svelte.dev/blog/frameworks-without-the-framework
[^2]: Svelte Team, "Svelte 5 is alive" — 2024-10-19. https://svelte.dev/blog/svelte-5-is-alive
[^3]: Svelte docs, "Runes". https://svelte.dev/docs/svelte/what-are-runes
[^4]: Svelte docs, "Migration guide". https://svelte.dev/docs/svelte/v5-migration-guide
[^5]: Rich Harris, "Svelte 3: Rethinking reactivity" — 2019-04-22. https://svelte.dev/blog/svelte-3-rethinking-reactivity
