# DefinitelyTyped/DefinitelyTyped

The community repository for high-quality TypeScript type definitions — the source of `@types/*` packages on npm.

## What it is

A monorepo of TypeScript type declaration files (`.d.ts`) for thousands of JavaScript libraries that ship without their own types. The `@types/*` namespace on npm is published from this repo: `npm install --save-dev @types/lodash` fetches types written here. Volunteer-maintained, community-curated, with strict review standards. Now somewhat in decline as more libraries ship types in-package, but still essential for legacy / untyped libraries.

## Key features

- Type definitions for ~7,000+ JS libraries (`@types/node`, `@types/lodash`, `@types/jquery`, etc.).
- Strict review process — types-only PRs get reviewed by maintainers + automated test suite.
- Automated publishing pipeline — merged PRs publish new `@types/*` versions automatically.
- Hacktoberfest-friendly.
- License terms vary per type-package (typically MIT-equivalent).

## Tech stack

- TypeScript primary (the type definitions themselves).
- Custom test infrastructure (`dtslint`, `dts-critic`) for type quality.

## When to reach for it

- You're using a JS library that doesn't ship its own types — check `@types/<library>` here.
- You're contributing type definitions for a community library.
- You're studying high-quality TypeScript declaration patterns.

## When *not* to reach for it

- The library you're using already ships its own types in-package (most modern libraries do).
- You want to handle types yourself via JSDoc + `checkJs`.

## Maturity signal

51k stars, 30k forks, last push very recent — actively maintained. 30k forks reflect contributor-PR workflow (each contributor forks). Volunteer maintainer model is durable; Microsoft has historically funded some of the core work.

## Alternatives

- Library-shipped types — most modern libraries include their own.
- JSDoc + `checkJs` — use when you don't want a TS file at all.

## Tags

typescript, types, library, awesome-list, javascript, community
