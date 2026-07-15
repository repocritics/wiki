# codex-team/editor.js

> A block-styled WYSIWYG editor whose output is clean JSON, not HTML — each block type is a separately-installed plugin.

[GitHub repo](https://github.com/codex-team/editor.js) ·
[Official website](https://editorjs.io) ·
[License: Apache-2.0](https://github.com/codex-team/editor.js/blob/next/LICENSE)

## Overview

Editor.js is a browser-based block editor built by CodeX, an open-source team[^1]. Instead of producing an HTML string like a classic rich-text editor, it serializes a document to a JSON array of typed blocks (`{ type, data }`), each rendered and saved by its own Tool plugin. The core ships almost no block types on its own — you install `@editorjs/header`, `@editorjs/list`, `@editorjs/image`, and so on separately and register them at construction. This is the project's defining design choice: the editor is a thin block-orchestration kernel, and nearly all user-visible capability lives in independently-versioned plugins.

The JSON-not-HTML stance is the second defining trait, and the source of its most common integration surprise. Editor.js gives you structured data that is easy to sanitize, store, diff, and re-render across surfaces (web, native apps, AMP, feeds, LLM pipelines) — but it does **not** ship an official way to turn that JSON back into display HTML. Rendering saved content is your responsibility. Teams that expect a "get HTML out" call the way TinyMCE or Quill work are frequently caught off guard.

It is popular (over 31k stars[^2]) and used widely for Notion-style editing surfaces in CMSes, comment systems, and knowledge tools. It is framework-agnostic vanilla TypeScript with direct DOM manipulation — no React/Vue dependency — which makes it embeddable anywhere but also means framework integrations (`react-editor-js` and friends) are community wrappers, not first-party.

## Getting Started

```bash
npm i @editorjs/editorjs @editorjs/header @editorjs/list
```

```javascript
import EditorJS from '@editorjs/editorjs';
import Header from '@editorjs/header';
import List from '@editorjs/list';

const editor = new EditorJS({
  holder: 'editorjs',            // id of a <div> on the page
  tools: {
    header: Header,
    list: List,
  },
});

// Serialize on demand — returns a Promise, not a string
const output = await editor.save();
```

The saved output is JSON, not markup:

```json
{
  "time": 1672531200000,
  "blocks": [
    { "type": "header", "data": { "text": "My title", "level": 2 } },
    { "type": "paragraph", "data": { "text": "Hello <b>world</b>" } }
  ],
  "version": "2.30.0"
}
```

Note that `data.text` can still contain inline HTML (bold, links, marker). To display saved content you convert this JSON to HTML yourself — per-block, on the client or server.

## Architecture / How It Works

The core manages block lifecycle, the toolbar, block navigation, and paste/sanitize plumbing. Everything with visible behavior is a **Tool**, and there are three kinds:

1. **Block Tools** — own a whole block (paragraph, header, image, list). A Block Tool is a class implementing `render()` (build the block's DOM) and `save()` (serialize its DOM back to `data`). The `render`/`save` pair is the whole data contract.
2. **Inline Tools** — formatting applied to a selection *inside* a block (bold, italic, link, marker). They mutate the block's inner HTML, which is why `data.text` carries HTML fragments.
3. **Block Tunes** — per-block modifiers surfaced in the block's settings menu (alignment, "move up", delete). Tunes attach extra state to a block without being a block themselves.

Each editable block is backed by a `contenteditable` region. There is no virtual DOM and no document schema in the ProseMirror sense — the "model" is just whatever JSON each Tool's `save()` chooses to emit, and the editor does not validate cross-block structure. On `save()`, the core walks every block and calls its Tool's `save()`, then runs the declared **sanitizer** config to strip disallowed tags. Sanitization is per-Tool and opt-in via each Tool's `sanitize` getter; the core does not globally scrub arbitrary fields.

Undo/redo is **not** in the core — it is a separate plugin (`editorjs-undo`). Real-time collaborative editing is on the published roadmap[^3] but has been unshipped for years; there is no built-in OT/CRDT layer today. The development default branch is `next`, and the changelog lives in-repo[^4].

## Production Notes

**Rendering output is your problem.** There is no first-party JSON→HTML renderer. Options: a community library like `editorjs-html`, a per-block renderer you write server-side, or rendering the blocks directly in your frontend framework. Budget for this — it is the single most common "wait, where's the HTML?" moment.

**Every block type is a separate dependency.** A realistic editor pulls in 8–15 `@editorjs/*` packages, each versioned independently of the core. Version skew between a Tool and the core API is a recurring source of breakage; pin versions and upgrade the core and tools together. Official tools (under the `editor-js` org) are maintained; community tools vary widely in quality and `readOnly` support.

**Data-shape migrations are on you.** A block's `data` schema is defined entirely by its Tool. If a Tool changes its shape across versions (e.g. `@editorjs/list` moving to a nested structure), previously-saved JSON may no longer round-trip. There is no schema/versioning framework — you own migrations of stored content.

**Server-side rendering.** It is a DOM/`contenteditable` editor and needs `window`; it cannot be instantiated during SSR. In Next.js and similar, import it dynamically / client-only.

**Sanitization is not automatic on read.** Because inline HTML is stored inside block `data`, re-sanitize on the way *out* if you render saved content into a page — the on-save sanitizer only covers tags each Tool declared, and does not protect a downstream renderer from stored markup.

**contenteditable quirks persist.** Paste normalization, mobile browsers, and IME composition (CJK input) carry the usual `contenteditable` edge cases. Test paste-from-Word and mobile flows explicitly.

## When to Use / When Not

**Use when:**
- You want structured, portable content (JSON) rather than an HTML blob to store and re-render across multiple surfaces.
- You want a Notion-style block UI without building block orchestration yourself.
- You need a framework-agnostic editor you can drop into vanilla, React, or Vue.
- Your content model maps cleanly to discrete, typed blocks.

**Avoid when:**
- You need real-time collaborative editing today — it is not built in.
- You need HTML output with zero conversion work — Quill/TinyMCE are closer to that.
- You need a rigorously-validated document schema or a rich programmable model — ProseMirror/Lexical fit better.
- Your content is free-form long-form prose that resists a strict block decomposition.

## Alternatives

- ProseMirror/prosemirror — use instead when you need a strict document schema and first-class collaborative editing (OT), and are willing to build more yourself.
- ueberdosis/tiptap — use instead when you want ProseMirror's model with a friendlier, headless API and ready-made extensions.
- quilljs/quill — use instead when you want a classic toolbar WYSIWYG with a Delta model and straightforward HTML.
- facebook/lexical — use instead when you're in React and want an extensible, high-performance editor framework from Meta.
- ianstormtaylor/slate — use instead when you want a fully controllable React-native editor model and accept assembling the UI yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| codex-editor (pre-2.0) | 2015–2018 | Early prototype under the original name[^4]. |
| 2.0 | 2019 | Full rewrite: TypeScript, plugin-per-block Tools API[^4]. |
| 2.19 | 2020 | Read-only mode. |
| 2.20 | 2021 | i18n and RTL support. |
| 2.27+ | 2022–2023 | Vertical/unified toolbar, nested Block Tunes menus. |
| 2.30 | 2024 | Ongoing 2.x line; unified toolbar work continuing[^3]. |

(Minor-version dates approximate; consult the in-repo changelog for exact releases.[^4])

## References

[^1]: About CodeX. https://codex.so
[^2]: GitHub repository — codex-team/editor.js (stars/forks/license as of fetch). https://github.com/codex-team/editor.js
[^3]: Editor.js roadmap (README) — collaborative editing and unified toolbar. https://github.com/codex-team/editor.js#roadmap
[^4]: Editor.js changelog. https://github.com/codex-team/editor.js/blob/next/docs/CHANGELOG.md

## Tags

typescript, javascript, wysiwyg, block-editor, rich-text-editor, json, contenteditable, cms, ui-component, plugin-architecture
