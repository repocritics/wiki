# tinacms/tinacms

> A Git-backed headless CMS that edits the Markdown/MDX files already in your repo, with an optional in-page visual editor.

[GitHub repo](https://github.com/tinacms/tinacms) ·
[Official website](https://tina.io) ·
[License: Apache-2.0](https://github.com/tinacms/tinacms/blob/main/LICENSE)

## Overview

TinaCMS is a headless content management system whose distinguishing choice is that content lives in your own Git repository as Markdown, MDX, JSON, or YAML — not in a proprietary database. Editors change files; commits land in the repo. It grew out of Forestry.io, a Git-based CMS that SSW shut down in 2023 and rebuilt as Tina[^1] (the `forestry` topic on the repo is a remnant of that lineage). The project is developed primarily by SSW.

The two halves that make Tina distinct are the **content API** and the **visual editor**. Tina compiles a schema you define into a GraphQL API over your content files, so instead of parsing frontmatter by hand you query `post.author.firstName` with typed results and cross-document references. On top of that it offers an opt-in live preview: an editor sidebar and click-into-the-page editing that writes back to the underlying files, aimed at making Markdown approachable for non-developers.

The defining tension is that the editing experience is open source and runs locally, but the collaborative production story is not free-standing. Real multi-editor use — authentication, media storage, an indexed content query layer, and editorial workflow — is designed around TinaCloud, the paid hosted backend. A fully self-hosted deployment is supported and documented, but it means standing up a data layer (a database plus an auth provider) yourself. Evaluate Tina on whether you accept that split, not on the local demo alone.

## Getting Started

Scaffold a starter (Next.js by default) or add Tina to an existing site:

```bash
npx create-tina-app@latest
# or, in an existing project:
npx @tinacms/cli@latest init
```

Define your content shape in `tina/config.ts`:

```ts
import { defineConfig } from "tinacms";

export default defineConfig({
  branch: process.env.HEAD || "main",
  clientId: process.env.NEXT_PUBLIC_TINA_CLIENT_ID, // TinaCloud (optional in local mode)
  token: process.env.TINA_TOKEN,
  build: { outputFolder: "admin", publicFolder: "public" },
  media: { tina: { mediaRoot: "uploads", publicFolder: "public" } },
  schema: {
    collections: [
      {
        name: "post",
        label: "Posts",
        path: "content/posts",
        format: "mdx",
        fields: [
          { type: "string", name: "title", label: "Title", isTitle: true, required: true },
          { type: "rich-text", name: "body", label: "Body", isBody: true },
        ],
      },
    ],
  },
});
```

```bash
npx tinacms dev -c "next dev"   # runs a local GraphQL server + your framework
# edit at http://localhost:3000/admin
```

## Architecture / How It Works

Tina has three moving parts that are easy to conflate:

1. **The schema and CLI.** `tina/config.ts` declares collections and fields. The `@tinacms/cli` compiles this into a typed GraphQL schema and a generated client (`tina/__generated__/`). This codegen step is the source of truth; forgetting to re-run it after a schema change is the most common early confusion.
2. **The content API (data layer).** GraphQL queries do not read files directly at request time — they read an **index**. In local dev the CLI builds this index in memory from the filesystem. In production the index lives in a database: TinaCloud manages it for you, or a self-hosted deployment points Tina at MongoDB (or another supported store) that you keep in sync with the repo. This indexing layer is what makes references and list queries fast, and it is the part people underestimate.
3. **The editor (React).** The admin UI and the optional in-page visual editing are React components. Visual editing works by wrapping your query data with `useTina`, which subscribes the page to editor changes and re-renders live. On save, edits are serialized back to Markdown/MDX/JSON and committed through the backend to Git.

Content is authored in a rich-text field stored as MDX and represented internally as an AST, which is how Tina round-trips between the visual editor and the source file without lossy Markdown reserialization. Framework support is broadest for Next.js (the starters and docs assume it), but the GraphQL client is framework-agnostic and Tina is used with Astro, Hugo, and others.

## Production Notes

- **You cannot skip the data layer in production.** The local filesystem index does not exist on a serverless deploy. You either point `clientId`/`token` at TinaCloud or you run the self-hosted backend with your own database and auth. Teams that prototype in local mode and then discover this at deploy time are a recurring support pattern.
- **Self-hosting is real but not turnkey.** The self-hosted path wires a Next.js API route to a `TinaNodeBackend`, a database adapter (MongoDB via the Vercel KV / Mongo levels), a Git provider, and an auth provider (Auth.js/Clerk are the documented options). It works, but it is several integrated services, not a single container.
- **Content and code share a repo and a review flow.** Editorial changes arrive as commits/PRs. This is a feature for developer-run sites and a friction point for non-technical editors who expect a Notion-like save button; branch-based editorial workflow exists on TinaCloud to soften this.
- **Codegen drift.** The generated client and types must match the schema. In CI, run the Tina build before your framework build so stale generated code doesn't ship. Committing `tina/__generated__` vs. regenerating it is a team decision with tradeoffs.
- **Rich-text is MDX, not arbitrary HTML.** Custom components in the body must be registered as templates in the schema to be editable; unregistered JSX or raw HTML can render but won't be first-class in the visual editor, and can surprise you on round-trip.
- **Scale of content.** Because queries hit an index rather than the filesystem, very large content sets are bounded by your data-layer store and its indexing time, not by Git. Plan the database sizing, not the repo.

## When to Use / When Not

**Use when:**
- Your content is (or should be) Markdown/MDX in Git and you want that to stay true.
- Developers own the site and want typed GraphQL access to content plus optional visual editing for writers.
- You are on Next.js or another JS framework and want a CMS that versions content alongside code.
- You want to avoid lock-in to a proprietary content database.

**Avoid when:**
- Your editors expect a fully hosted, zero-infrastructure SaaS and you don't want to pay for TinaCloud or run the self-hosted stack.
- Content is highly relational or high-volume in ways better served by a database-native CMS.
- You need many concurrent non-technical editors with rich workflow and don't want everything mediated through Git commits.
- Your stack is not JavaScript and you don't want a Node GraphQL layer in the mix.

## Alternatives

- decaporg/decap-cms — Git-based CMS (formerly Netlify CMS); simpler and fully free, but no typed GraphQL layer and a less polished editor.
- sveltia/sveltia-cms — a lighter, modern Decap-compatible Git CMS; use when you want Git-backed editing without Tina's data layer.
- strapi/strapi — database-backed self-hosted headless CMS; use when content is relational and you don't want it in Git.
- payloadcms/payload — code-first TypeScript headless CMS with its own database and admin; use when you want a full app backend, not file-based content.
- sanity-io/sanity — hosted structured-content platform with real-time collaboration; use when editor experience and collaboration outweigh keeping content in Git.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-07 | Repository created; early Tina as an inline React editing toolkit[^2]. |
| — | 2021 | Tina Cloud beta; pivot toward Git-backed content API + hosted backend. |
| 1.0 | 2022 | GraphQL content API and schema-driven model stabilized as the core product[^3]. |
| — | 2023 | Forestry.io sunset; users migrated to Tina[^1]. |
| — | 2024–2026 | Self-hosted data layer, MDX rich-text, and framework support matured; actively maintained by SSW (last push 2026-07). |

## References

[^1]: "Forestry is now TinaCMS" / Forestry sunset announcement, TinaCMS blog. https://tina.io/blog
[^2]: TinaCMS repository, created 2019-07-23. https://github.com/tinacms/tinacms
[^3]: TinaCMS documentation — schema, content API, and self-hosting. https://tina.io/docs/

## Tags

typescript, headless-cms, git-based-cms, markdown, mdx, graphql, react, nextjs, jamstack, content-management, visual-editing
