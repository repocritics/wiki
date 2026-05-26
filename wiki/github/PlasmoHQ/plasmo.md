# PlasmoHQ/plasmo

> The browser extension framework — Plasmo is to browser extensions what Next.js is to web apps.

[GitHub repo](https://github.com/PlasmoHQ/plasmo) ·
[Official website](https://www.plasmo.com) ·
[License: MIT](https://github.com/PlasmoHQ/plasmo/blob/main/LICENSE)

## Overview

Plasmo is a framework for building browser extensions targeting Chrome, Firefox, Edge, Brave, Opera, and Safari from a single TypeScript codebase[^1]. It abstracts the Manifest V3 background script / content script / popup boundaries into a file-system convention (à la Next.js), ships a dev server with hot module reload (including for content scripts), and handles the cross-browser build matrix.

The framework grew in relevance with the **Manifest V3 transition** (Chrome's deprecation of MV2 in 2024) because of how much harder MV3 made hand-rolled extension development: service workers instead of long-running background pages, restricted `eval`, stricter CSP, network request interception via `declarativeNetRequest`. Plasmo absorbs most of that complexity.

As of 2025 the OSS framework's maintenance cadence has slowed somewhat as Plasmo Corp focuses on **Itero** (their hosted in-extension distribution service)[^2]. The core remains widely used in production extensions; community alternatives (WXT) have gained ground.

## Getting Started

```bash
pnpm create plasmo my-extension
cd my-extension
pnpm dev
```

Load the unpacked extension from `build/chrome-mv3-dev` in `chrome://extensions` (Developer Mode → Load unpacked).

A minimal popup:

```tsx
// popup.tsx
import { useState } from "react"

function IndexPopup() {
  const [count, setCount] = useState(0)
  return (
    <div style={{ padding: 16, minWidth: 240 }}>
      <button onClick={() => setCount(c => c + 1)}>
        Clicked {count} times
      </button>
    </div>
  )
}

export default IndexPopup
```

A content script:

```ts
// contents/highlight.ts
import type { PlasmoCSConfig } from "plasmo"

export const config: PlasmoCSConfig = {
  matches: ["https://*.github.com/*"]
}

document.body.style.outline = "2px solid magenta"
```

A background service worker:

```ts
// background.ts
chrome.runtime.onInstalled.addListener(() => {
  console.log("installed")
})
```

Build for production across browsers:

```bash
pnpm build --target=chrome-mv3
pnpm build --target=firefox-mv3
pnpm build --target=safari-mv3
```

## Architecture / How It Works

Plasmo uses Parcel under the hood as its bundler[^3]. The framework provides:

1. **File-system manifest generation.** Files named `popup.tsx`, `options.tsx`, `background.ts`, `contents/*.ts`, `newtab.tsx` automatically populate the right slots in `manifest.json`. Page-style sidepanels (`sidepanel.tsx`) and bookmark pages are supported similarly.
2. **React (default) + Vue / Svelte / Solid support.** Choose at `create plasmo` time. React is the best-supported path.
3. **Storage abstraction (`@plasmohq/storage`).** Wraps `chrome.storage.local` / `chrome.storage.sync` with a hook-friendly React API and cross-context sync.
4. **Messaging abstraction (`@plasmohq/messaging`).** Typed message passing between content scripts, background, popup — replaces raw `chrome.runtime.sendMessage`.
5. **HMR for content scripts.** Live reloads injected content without a full extension reload — significantly faster dev loop than vanilla MV3 development.
6. **Per-browser manifest delta.** A single source manifest is transformed for each target (Firefox `browser_specific_settings`, Safari background scripts, etc.).

**Manifest V3 semantics** that Plasmo abstracts:
- Background **service worker** instead of persistent page. Lifecycles in minutes; state in `chrome.storage` (or restoring via `chrome.alarms`).
- `host_permissions` separate from `permissions`. Plasmo's manifest config keeps them distinct.
- `declarativeNetRequest` for ad-blocker-style request modification (replacing `webRequestBlocking`).
- Restricted `executeScript` requires `world: "MAIN"` for page-context injection.

## Production Notes

**Service worker lifecycle.** The biggest MV3 footgun. Service workers can be killed at any time after ~30 seconds idle. Patterns that bit production extensions:
- `setTimeout` longer than 30 seconds is unreliable — use `chrome.alarms`.
- Global variables in the SW don't survive restarts — persist to `chrome.storage`.
- WebSocket / long-poll connections will be killed. Re-establish on demand.
- Plasmo's `@plasmohq/messaging` makes message handling idempotent across restarts; relying on raw `chrome.runtime.connect` ports is fragile.

**Content script isolation.** Content scripts run in an isolated world by default — they share the page DOM but not its JavaScript globals. To touch page globals, inject a `<script>` tag or use `world: "MAIN"`. Plasmo's `inject/` folder convention handles main-world scripts.

**Store review caveats.**
- Chrome Web Store rejects extensions with broad `host_permissions: ["<all_urls>"]` for new submissions in many categories. Narrow to specific hosts.
- Firefox requires source code submission for non-MIT/Apache packaged extensions; the build output must reproduce from source.
- Safari requires Xcode + a Mac developer account; Plasmo generates the Safari wrapper but you still package via Xcode.
- Review times: Chrome 1–7 days, Firefox 1–14 days, Safari (Apple) 1–14 days, Edge 1–7 days.

**Itero distribution.** Plasmo Corp offers Itero as an alternative distribution channel — install extensions directly from a hosted URL without store review. Useful for B2B / internal tools. The OSS framework works fine without Itero.

**Maintenance status.** Issue response times have slowed since 2024-Q3. The framework is in "feature-complete maintenance" mode rather than active development. Critical bug fixes still ship. For greenfield projects, evaluate WXT (Vite-based) and `@crxjs/vite-plugin` as well.

**Bundle size.** Plasmo extensions weigh roughly 200–500 KB unzipped for a simple React popup. Heavy dependencies (Chart.js, three.js) add up quickly — extensions have a soft 10 MB ZIP limit before review friction.

**Permissions UX.** Users see permission lists on install. Optional permissions (`chrome.permissions.request`) can be requested at runtime — better install conversion than asking for everything upfront.

## When to Use / When Not

**Use when:**
- Building a cross-browser extension (Chrome + Firefox + Edge) from one codebase.
- You want React/Vue/Svelte components in popup / options / sidepanel.
- You want HMR for content scripts (the dev productivity win is real).
- You're shipping an MV3 extension and don't want to hand-roll the service-worker lifecycle plumbing.

**Avoid when:**
- You're shipping a tiny single-file extension and the framework adds more than it solves.
- You need Vite-based HMR and ecosystem (WXT is the modern alternative).
- The maintenance status of Plasmo OSS makes you uneasy — confirm activity on GitHub before committing.
- You need very specific Manifest V3 features Plasmo doesn't yet abstract (some `declarativeNetRequest` advanced patterns).

## Alternatives

- **WXT** — Vite-based, similar file-system conventions, more active maintenance in 2025.
- **`@crxjs/vite-plugin`** — Vite plugin only; you assemble the framework yourself.
- **Hand-rolled webpack + manifest** — full control, full burden.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2022-03 | Initial release, Parcel-based. |
| 0.50 | 2022-08 | Storage / messaging APIs stable. |
| 0.70 | 2023-01 | Multi-browser targets, Firefox support. |
| 0.80 | 2023-06 | Itero launch, MV3 polish. |
| 0.85+ | 2024 | MV3 service worker improvements. |
| 0.88+ | 2024–2025 | Maintenance cadence slowed; security fixes only on many issues. |

## References

[^1]: Plasmo docs, "Introduction". https://docs.plasmo.com
[^2]: Plasmo blog, "Itero — in-extension distribution". https://www.plasmo.com/itero
[^3]: Plasmo source, bundler architecture. https://github.com/PlasmoHQ/plasmo
