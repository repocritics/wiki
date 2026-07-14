# alpinejs/alpine

> A minimal framework for composing reactive behavior directly in server-rendered HTML, using markup attributes instead of a build step.

[GitHub repo](https://github.com/alpinejs/alpine) ·
[Official website](https://alpinejs.dev) ·
[License: MIT](https://github.com/alpinejs/alpine/blob/main/LICENSE.md)

## Overview

Alpine.js is a small client-side framework created by Caleb Porzio (author of Laravel Livewire) and first released in late 2019[^1]. It occupies the gap between hand-written vanilla JavaScript and a full SPA framework: you keep rendering HTML on the server, then annotate elements with `x-` attributes to add reactivity, event handling, and conditional rendering. There is no virtual DOM diff of the whole page, no client router, and — by design — no required build tooling. A single `<script defer>` tag from a CDN is a complete install.

The defining tradeoff is **logic-in-markup**. Alpine expressions live as strings inside HTML attributes (`x-on:click="open = !open"`), evaluated at runtime. This is what makes Alpine approachable and buildless, and it is also its ceiling: expressions are not type-checked, not tree-shaken, awkward to unit-test, and evaluated through a mechanism that trips strict Content-Security-Policy setups. Alpine is deliberately not trying to be React; it is the interactivity layer for server-driven stacks (Livewire, Laravel Blade, Rails/Hotwire-adjacent, Django templates, htmx).

Alpine v3 (2021) was a ground-up rewrite that introduced a plugin architecture and replaced the bespoke reactivity engine with Vue's standalone `@vue/reactivity` package[^2]. Most Alpine code you see today is v3.

## Getting Started

```html
<!-- CDN: defer is required so Alpine initializes after the DOM is parsed -->
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>

<div x-data="{ open: false, count: 0 }">
  <button x-on:click="open = !open">Toggle</button>

  <div x-show="open" x-transition>
    <p x-text="'clicked ' + count + ' times'"></p>
    <button @click="count++">+1</button>
  </div>
</div>
```

Or as an npm module, where you register data/plugins before starting Alpine:

```bash
npm install alpinejs
```

```js
import Alpine from 'alpinejs'

Alpine.data('dropdown', () => ({
  open: false,
  toggle() { this.open = !this.open },
}))

window.Alpine = Alpine
Alpine.start()
```

## Architecture / How It Works

On startup Alpine walks the DOM, finds each `x-data` element as a **component root**, evaluates its object literal into reactive state, and initializes the descendant directives inside that scope. A `MutationObserver` watches for later DOM changes so dynamically inserted markup is initialized automatically. There is no compile step — everything happens in the browser at runtime.

**Reactivity** in v3 is `@vue/reactivity`[^2]: state objects become `Proxy`-wrapped, and each directive registers an *effect* that re-runs when the state it reads changes. This gives fine-grained, per-directive updates rather than whole-tree re-rendering.

**Expressions** are the coupling story. Each `x-` attribute value is compiled into a function with the component scope injected via a `with`-style binding. In the default build this uses runtime function construction (`eval`-class evaluation), which is why Alpine ships a separate **CSP build** with a restricted expression evaluator for environments that forbid `unsafe-eval`[^3] — at the cost of a more limited expression syntax.

Core building blocks:

- **Directives** — `x-data`, `x-init`, `x-show`, `x-if`, `x-for`, `x-bind` (`:`), `x-on` (`@`), `x-model`, `x-text`, `x-html`, `x-transition`, `x-ref`, `x-effect`, `x-cloak`, `x-teleport`, `x-ignore`.
- **Magics** — `$el`, `$refs`, `$store`, `$watch`, `$dispatch`, `$nextTick`, `$root`, `$data`, `$id`.
- **Global APIs** — `Alpine.data()` (reusable components), `Alpine.store()` (global reactive state), `Alpine.directive()` / `Alpine.magic()` (extension points), `Alpine.bind()`.
- **Official plugins** — `mask`, `intersect`, `persist`, `focus`, `collapse`, `morph`, `history`, plus the separate Alpine UI headless components.

Extension registration must happen inside the `alpine:init` event (or before `Alpine.start()`), because Alpine only reads registered data/directives once at boot.

## Production Notes

**CSP is the recurring footgun.** The default build's expression evaluation requires `unsafe-eval`. Locked-down applications must either loosen CSP (usually unacceptable) or adopt the CSP build and rewrite expressions to its narrower grammar — you cannot mix arbitrary inline JS. Audit this before committing Alpine to a security-sensitive app.

**No build step means no safety net.** Expressions are untyped strings in HTML: a typo in `x-text="user.naem"` fails silently at runtime, IDEs give little help, and there is no tree-shaking. Logic-heavy components become hard to test because the behavior lives in attributes rather than in importable functions. Push non-trivial logic into `Alpine.data()` factories so it is at least real JavaScript.

**Reactivity gotchas.** State is only reactive through the proxy. Destructuring state (`let { open } = this`) or capturing a primitive into a plain variable severs reactivity. Deep mutation works, but replacing an object reference behaves differently from mutating it, and closures over stale values are a common source of "why didn't the UI update" bugs.

**SPA-style navigation.** Alpine initializes on page load and via its `MutationObserver`, but full client-side page swaps (Turbo, some htmx flows) can leave stale or double-initialized components. Loading the Alpine script twice throws. Coordinate with the navigation library — the `morph` plugin exists partly to reconcile server-swapped HTML with live Alpine state.

**Scaling ceiling.** Alpine is excellent for hundreds of small, independent islands of interactivity. It is a poor fit for a single large interconnected client state graph: there is no store selector model beyond `Alpine.store`, no component-tree devtools on par with React/Vue, and shared cross-component state gets unwieldy fast.

**Funding/maintenance.** Alpine is maintained by Caleb Porzio and sponsors rather than a corporation. Development pace is steady but small-team; the repo is actively maintained.

## When to Use / When Not

**Use when:**
- Your HTML is rendered server-side (Livewire, Blade, Rails, Django, Laravel/htmx) and you want dropdowns, modals, tabs, and toggles without a JS framework.
- You want zero build tooling — a script tag and attributes.
- The interactivity is many small, independent widgets rather than one large app.

**Avoid when:**
- You need a real SPA: client routing, deep component trees, large shared state, code-splitting.
- You operate under a strict CSP and cannot adopt the restricted CSP build.
- You want type safety, testable component logic, or IDE-grade refactoring of your view logic.

## Alternatives

- bigskysoftware/htmx — hypermedia-driven; use instead when interactivity is mostly server round-trips and you want almost no client state.
- hotwired/stimulus — modest class-based JS controllers for server HTML; use when you prefer logic in real JS files over inline attributes.
- vuejs/petite-vue — Vue's own ~6KB Alpine-style progressive-enhancement build; use when you already think in Vue templates.
- sveltejs/svelte — compiled component framework; use when you need a real build and component model, not sprinkles.
- livewire/livewire — server-driven Laravel components that bundle Alpine; use when your stack is Laravel and you want server-rendered reactivity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019-11 | Initial release by Caleb Porzio; `x-` directives, buildless[^1]. |
| 2.0 | 2020 | Reactivity and directive refinements; last v2 is 2.8.2. |
| 3.0 | 2021-11 | Full rewrite: plugin architecture, `@vue/reactivity`, `Alpine.data`/`store`/`magic`/`directive` APIs, async expressions[^2]. |

## References

[^1]: Caleb Porzio announced Alpine.js in late 2019; the repository was created 2019-11-28. Porzio is also the author of Laravel Livewire, for which Alpine is the client-side companion. https://alpinejs.dev
[^2]: Alpine v3 uses Vue's standalone reactivity core, published as `@vue/reactivity`. https://www.npmjs.com/package/@vue/reactivity
[^3]: Alpine ships a dedicated CSP-safe build with a restricted expression evaluator for environments that disallow `unsafe-eval`. https://alpinejs.dev/advanced/csp

## Tags

javascript, html, frontend, reactive, progressive-enhancement, no-build, spa-alternative, laravel, livewire, dom, minimal
