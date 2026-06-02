# vuejs/vue

The end-of-life Vue 2 codebase. New projects should use `vuejs/core` (Vue 3); this repo is archived in spirit.

## What it is

The original Vue.js progressive JavaScript framework that defined the project from 2014 through Vue 2's release cycle. The maintainers formally end-of-lifed Vue 2 on 2023-12-31. The package is still distributed on CDNs and npm for backwards compatibility, but no new features, updates, or fixes are landing here. The actively maintained Vue lives at `vuejs/core` (Vue 3); the upgrade path is documented at `v3-migration.vuejs.org`.

## Key features

- The original Options API and template syntax that made Vue famous.
- Reactive data binding via the Vue 2 reactivity system (Object.defineProperty-based — not the Proxy-based system Vue 3 uses).
- Single-file components (`.vue`) supported via `vue-loader` in the Vue 2 toolchain.
- Slot composition, mixins, scoped CSS, and the directive system inherited by Vue 3 (with semantic adjustments).

## Tech stack

- TypeScript at the source level, compiled JavaScript at distribution.
- Custom build via Rollup with internal `scripts/` tooling.
- npm package: `vue` (the package name now ships Vue 3 — Vue 2 is at `vue@2`).

## When to reach for it

- You're maintaining a Vue 2 project that can't be migrated and need to track the EOL branch.
- You're paying for the HeroDevs "Vue 2 NES" (Never-Ending Support) commercial security support and need the underlying source for reference.
- You're studying the evolution of mainstream JS reactivity systems and want the pre-Proxy implementation as reference.

## When *not* to reach for it

- You're starting a new project — go to `vuejs/core` (Vue 3).
- You need security patches for production deployments — the EOL is real; only the commercial HeroDevs offering provides ongoing patches.
- You want active community support — the Vue community has migrated to Vue 3 for documentation, plugins, and Q&A.

## Maturity signal

209k stars, 33k forks, MIT — historically one of the most-starred frontend frameworks. End-of-Life 2023-12-31. Last push October 2024 (likely the final security-window backport before EOL took full effect). This is "abandoned" by intent rather than by accident — the project's lifecycle ended cleanly with a documented migration path.

## Alternatives

- `vuejs/core` — the canonical replacement; Vue 3 with Composition API, Proxy-based reactivity, and active maintenance.
- `nuxt/nuxt` — use when you want the meta-framework on top of Vue 3.
- HeroDevs Vue 2 NES — commercial support if migration isn't feasible and you need ongoing security patches.

## Notes

The "still distributed on CDNs" caveat matters — projects pinning `vue@^2` will continue to receive package installs but no patches, including for any future CVEs disclosed in Vue 2's reactive system or template compiler. The migration to Vue 3 is the official answer; the commercial NES support is the bridge for organizations that can't migrate. License (MIT) means the codebase can be forked and patched independently if needed.

## Tags

vue, javascript, typescript, web-framework, frontend, user-interface, component-based, end-of-life, legacy
