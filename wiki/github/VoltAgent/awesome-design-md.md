# VoltAgent/awesome-design-md

A curated collection of ready-to-use DESIGN.md files extracted from well-known brand websites, so AI coding agents can generate UI matching a chosen design language.

## What it is

This repository is a curated set of DESIGN.md documents — plain-Markdown design-system specs — extracted from real, developer-focused and consumer brand websites. DESIGN.md is a format introduced by Google Stitch: a single Markdown file an AI agent reads to generate visually consistent UI, with no Figma exports, JSON schemas, or extra tooling. You copy one file into your project root, tell your coding agent to build a page that looks like it, and get output that stays consistent with that brand's visual language. It is aimed at developers, designers, and "vibecoders" generating UI with AI agents.

## Key features

- A browsable collection of DESIGN.md files organized by category (AI/LLM platforms, developer tools, backend/database/DevOps, productivity/SaaS, design/creative, fintech/crypto, e-commerce, media, automotive), plus a "Retro Web" series of 1990s-era designs.
- Each site entry ships three files: `DESIGN.md` (what agents read), `preview.html`, and `preview-dark.html` (visual catalogs of swatches, type scale, buttons, and cards in light and dark surfaces).
- A consistent nine-section structure per file: visual theme, color palette with roles, typography rules, component stylings, layout principles, depth/elevation, do's and don'ts, responsive behavior, and an agent prompt guide.
- Files follow the Google Stitch DESIGN.md specification, extended with additional sections, and are readable by AI coding agents and by Google Stitch directly.
- A request path for missing sites, including private requests.

## Tech stack

- No programming language or root build manifests; this is a content repository of Markdown (`DESIGN.md`) and static HTML preview files.
- MIT licensed.
- Uses the Google Stitch DESIGN.md format as its underlying specification.

## When to reach for it

- You want an AI agent to generate UI that matches a recognizable brand's look without hand-writing a design spec.
- You want a drop-in, tooling-free design brief (a single Markdown file) rather than Figma exports or token JSON.
- You want side-by-side light/dark preview catalogs to sanity-check a design language before using it.
- You are prototyping or "vibe-coding" a landing page and want a consistent starting aesthetic.

## When *not* to reach for it

- You need a precise, machine-readable design-token pipeline tied to your own source of truth; these files are extracted approximations of publicly visible CSS values, provided "as is."
- Your project has its own distinct design language not represented in the collection.
- You need runtime theming, component code, or a design system library — this repository ships design descriptions, not implementations.
- You require guaranteed fidelity to a brand's official identity; the repo explicitly does not claim ownership of any site's visual identity.

## Maturity signal

This is an actively maintained curated content collection: created in late March 2026, last pushed in mid-June 2026 (about a month before this page's facts), MIT licensed, and not archived, with a very high star count and an Awesome-list badge. For a content repo, cadence naturally tracks new additions rather than code releases; the recent push and large, categorized catalog indicate ongoing curation. The README claims a top-150 global GitHub ranking.

## Alternatives

- **Google Stitch** — use it when you want to generate or author a DESIGN.md from your own site or mockups rather than reuse an extracted brand file; Stitch originates the format.
- **Figma / design-token exports** — use these when you need precise, machine-readable tokens tied to your own design source instead of extracted, approximate values.
- **A hand-written project DESIGN.md** — use this when your product has a unique visual identity that no brand file in the collection matches.

## Notes

- The collection separates `DESIGN.md` (how a project should look, read by design agents) from `AGENTS.md` (how to build a project, read by coding agents), framing them as complementary files.
- The extracted tokens represent publicly visible CSS values and are provided without warranty; the repository is a reference, not an official brand asset.

## Tags

awesome-list, design-system, design-tokens, design-md, ui, frontend, markdown, ai-agents, vibe-coding
