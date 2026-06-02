# typescript-cheatsheets/react

The canonical community reference for using React with TypeScript — answers the "how do I type X in React?" question with battle-tested patterns.

## What it is

A community-maintained collection of cheatsheets covering the practical patterns of typing React code in TypeScript. Started as a single Gist by Chenglou and grew into a Docusaurus site with multiple sub-cheatsheets. The most-cited resource when React+TS developers hit "how do I properly type this prop / hook / HOC?" Authored under an open governance model by the typescript-cheatsheets organization with ongoing contributions from the React-Typescript community.

## Key features

- Multi-page cheatsheet structure: basic, advanced, migrating-from-JS, HOC patterns, useful-hooks, and how-to-contribute.
- Side-by-side "do this / don't do this" code examples for nearly every pattern.
- Migrating-from-JS-to-TS guide that walks through real-world conversion steps.
- Coverage of advanced topics: discriminated unions, conditional types in props, polymorphic components, generic hooks.
- Multi-language translations (Chinese, Japanese, Korean, Russian, Brazilian Portuguese, Spanish, Turkish, others).
- MIT-licensed — safe to vendor patterns into internal style guides.

## Tech stack

- Markdown content as the canonical source.
- Docusaurus for the published site.
- TypeScript at the language tag for the embedded code examples.

## When to reach for it

- You're a React developer adopting TypeScript and want a pattern reference rather than a textbook.
- You're a tech lead defining team conventions for React+TS components.
- You hit a "how do I type X" problem and want the community-vetted approach before rolling your own.
- You're migrating a JS React codebase to TS file-by-file and need a sequencing guide.

## When *not* to reach for it

- You're new to TypeScript itself — start with the official handbook first; this cheatsheet assumes TS fluency.
- You want a deep theoretical TS course — `total-typescript` (paid) and `type-challenges` are closer-fit.
- You want React without TS — there are plenty of plain-JS React tutorials.

## Maturity signal

Actively maintained for 6+ years. Open-issues count stays low because the maintainers triage aggressively. The translation track + Docusaurus migration signal that this isn't a single-author cheatsheet; it's a community asset. MIT license + clean contribution flow + no commercial product attached = stable target for vendoring patterns into internal docs.

## Alternatives

- React + TypeScript official docs — for the canonical type definitions.
- `total-typescript` paid courses — when you want video-driven structured learning.
- `type-challenges/type-challenges` — when you want to level up TS type-system fluency through puzzles.

## Notes

The cheatsheet is opinionated about modern React patterns (hooks-first, functional components, no class components in new code). Anyone maintaining a class-component codebase will find the migrating-from-JS guide more useful than the basic cheatsheet. The "useful-hooks" sub-cheatsheet is the most-pasted-into-codebases section — its type patterns for custom hooks have effectively become the React+TS convention.

## Tags

react, typescript, awesome-list, education, mit-license, learn-to-code, cheatsheet, conventions
