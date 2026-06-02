# chenglou/pretext

A fast, accurate, comprehensive text measurement and layout library — by Cheng Lou (Reason / ReScript core team).

## What it is

A TypeScript library for low-level text measurement and layout — width, height, line-breaking, kerning, font fallback, mixed-script handling. Aimed at canvas / WebGL / non-DOM renderers that can't rely on browser text layout. Authored by Cheng Lou, who has shipped multiple high-quality OSS libraries in adjacent layout / rendering spaces.

## Key features

- Accurate text measurement matching browser layout where applicable.
- Line-breaking, font fallback, mixed-script support.
- TypeScript-first, framework-agnostic.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Pure library — no UI dependencies.

## When to reach for it

- You're building canvas / WebGL UIs that need accurate text layout outside the DOM.
- You're implementing a custom renderer (PDF, native, embedded) where browser CSS isn't available.

## When *not* to reach for it

- You're using DOM-rendered text — browser layout is more capable and free.
- You don't need accurate measurement — `measureText` is enough for simple cases.

## Maturity signal

48k stars, 2.7k forks, MIT, actively maintained. Authored by Cheng Lou (known for ReasonReact and other layout work). The high star count for a niche library reflects the author's reputation in the JS/ReScript community.

## Alternatives

- `harfbuzz/harfbuzz` (via WASM) — use for the canonical text-shaping engine.
- Browser `measureText` — use when you only need width.

## Tags

typescript, library, text-layout, typography, canvas, rendering, mit-license
