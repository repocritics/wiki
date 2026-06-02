# airbnb/javascript

Airbnb's JavaScript style guide — one of the most-cited code-style references for JS teams.

## What it is

A markdown style guide originally authored at Airbnb that codifies opinions on JavaScript: variable declarations, arrow functions, destructuring, modules, naming, formatting, control flow, and ES6+ idioms. The guide is the canonical reference for the `eslint-config-airbnb` and `eslint-config-airbnb-base` packages, which most JS projects bootstrap from when adopting Airbnb's conventions. Covers vanilla JS, React patterns, and CSS-in-JS in companion documents.

## Key features

- Single-page style guide for JS plus separate guides for React, CSS-in-JS.
- Direct mapping to ESLint configurations (`eslint-config-airbnb`, `-base`, `-typescript`).
- Each rule has a brief example showing "do this" vs. "don't do this".
- Translated into ~20 languages by community contributors.
- MIT-licensed — clean for vendoring into internal style docs.

## Tech stack

- Markdown content for the guide.
- The companion ESLint packages are JS modules published to npm.

## When to reach for it

- You're standardizing JS style across a team and want a battle-tested reference to anchor on.
- You're onboarding new JS engineers and want a curated "modern JS conventions" document.
- You're configuring ESLint and want a pre-built config that's broadly known.

## When *not* to reach for it

- You're allergic to specific Airbnb opinions (semicolons, max-line-length, etc.) — picking individual rules from the guide is fine, but be aware "Airbnb style" is a recognizable signal that comes with all its choices.
- You're TypeScript-first and want TS-native conventions — `typescript-eslint` recommendations are closer-fit; Airbnb's TS support is a thin layer on top.
- You want a guide that tracks bleeding-edge proposals — the cadence is conservative.

## Maturity signal

148k stars, 27k forks, MIT, last push April 2026 — actively maintained on a deliberate cadence (Airbnb's frontend engineering team owns the project). 14-year-old project that survived the rise and shift of multiple JS frameworks. Open-issues count of 159 is low for the scope, reflecting tight ownership.

## Alternatives

- `prettier/prettier` — use when you want auto-formatting without per-rule opinions.
- `standard/standard` — use when you want a "no-config, no semicolons" minimalist style.
- `typescript-eslint/typescript-eslint` recommendations — use when TS is primary.

## Notes

The guide's most-debated entries (no semicolons, single-vs-double quotes, max line length, trailing commas) are also the ones most projects toggle. License (MIT) makes it safe to fork the guide as an internal company style doc. The package family (`eslint-config-airbnb*`) is the operational artifact; the markdown is the documentation surface.

## Tags

awesome-list, javascript, style-guide, eslint, code-quality, education, developer-tools, conventions
