# facebook/docusaurus

Meta's static-site generator for documentation — React-based, opinionated for technical docs, used by ~thousands of OSS projects' docs sites.

## What it is

A static site generator that turns Markdown / MDX documentation into a polished React-powered site. Originally built at Meta for the React ecosystem's docs, now widely adopted across OSS projects. Provides built-in versioning, i18n, search (Algolia DocSearch integration), and a familiar three-pane docs UI out of the box.

## Key features

- File-based routing — drop `.md` / `.mdx` files into the docs folder; URLs auto-derive.
- Docs versioning (v1, v2, v3 of the same docs side-by-side).
- i18n with Crowdin integration.
- Algolia DocSearch built-in.
- Blog + docs in the same site.
- MDX for interactive React components in markdown.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- React for the rendered output.
- Webpack-based build pipeline.

## When to reach for it

- You're standing up an OSS project's documentation site and want a battle-tested generator.
- You need versioned docs (multiple major versions of your library).
- You want i18n with a clear contribution flow.

## When *not* to reach for it

- You want a minimal-runtime site — Docusaurus ships React; lighter generators (Hugo, Zola, Astro) produce smaller output.
- You don't need React interactivity — a plain markdown renderer is sufficient.

## Maturity signal

65k stars, 10k forks, MIT, actively maintained by the Meta open-source team plus community.

## Alternatives

- VitePress — Vue-based alternative, popular in Vue/Vite ecosystem.
- Astro Starlight — use for a leaner, fully-tree-shaken docs site.
- mdBook — use for Rust-style chapter-based books.
- GitBook — use when you want a hosted, paid product.

## Tags

react, typescript, documentation, static-site-generator, frontend, mit-license, mdx, framework
