# ueberdosis/tiptap

> A headless, framework-agnostic rich-text editor toolkit built as a high-level API over ProseMirror.

[GitHub repo](https://github.com/ueberdosis/tiptap) ·
[Official website](https://tiptap.dev) ·
[License: MIT](https://github.com/ueberdosis/tiptap/blob/main/LICENSE.md)

## Overview

Tiptap is a rich-text editor framework for the web, maintained by the German studio überdosis and first released in 2019[^1]. It is "headless": it ships editing behavior and a document model but no UI, no toolbar, and no CSS. You bring the markup and styling; Tiptap manages the editable surface, the schema, commands, and the transaction pipeline. Bindings exist for React, Vue 2/3, Svelte, and plain JavaScript.

The defining fact about Tiptap is that it is a wrapper over [ProseMirror](https://prosemirror.net/)[^2]. Nearly every capability — the document model, transactions, decorations, node views — is ProseMirror's; Tiptap's value is a friendlier, TypeScript-first extension system and a chainable command API on top. This is the central tension: for simple editors the abstraction is pleasant and you never see ProseMirror, but for anything non-trivial the abstraction leaks and you must learn ProseMirror concepts (schema, plugins, `Transaction`, `Decoration`, node views) anyway. Tiptap is best understood as ergonomics over ProseMirror, not a replacement for understanding it.

Tiptap also runs an open-core business. The editor and the core extensions are MIT; a set of "Pro" extensions (collaborative comments, document versioning, AI, import/export/conversion) and the hosted Tiptap Cloud require a paid subscription[^3]. The open-source editor is fully usable standalone, but some ecosystem docs steer toward the commercial suite.

## Getting Started

```bash
npm install @tiptap/core @tiptap/pm @tiptap/starter-kit
# plus a framework binding, e.g. React:
npm install @tiptap/react
```

```tsx
// React — StarterKit bundles the common nodes/marks (paragraph, bold, lists, …)
import { useEditor, EditorContent } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";

export function Editor() {
  const editor = useEditor({
    extensions: [StarterKit],
    content: "<p>Hello world</p>",
    immediatelyRender: false, // avoid SSR hydration mismatch (Next.js)
  });

  return <EditorContent editor={editor} />;
}
```

The editor is unstyled — the rendered `.ProseMirror` element inherits nothing until you add your own CSS. `@tiptap/pm` is a peer package that bundles the underlying ProseMirror libraries at a single coherent version; installing it (rather than the raw `prosemirror-*` packages) is the supported path.

## Architecture / How It Works

ProseMirror is a set of low-level modules: `prosemirror-model` (a schema-constrained document tree), `prosemirror-state` (immutable editor state + transactions), `prosemirror-view` (the contenteditable DOM view), `prosemirror-transform` (position-mapped changes), and `prosemirror-commands`. Tiptap composes these and exposes three abstractions:

1. **Extensions** — the unit of everything. `Node` (block/inline content like paragraph, image), `Mark` (inline formatting like bold, link), and generic `Extension` (behavior with no schema, like keyboard shortcuts). Each extension contributes schema fragments, commands, input rules, and ProseMirror plugins. `StarterKit` is just a bundle of the common ones.
2. **Command chaining** — `editor.chain().focus().toggleBold().run()` composes multiple ProseMirror commands into a single transaction. This is the API most users interact with.
3. **Framework bindings** — `@tiptap/react`, `@tiptap/vue-3`, `@tiptap/vue-2`, `@tiptap/svelte` wrap the framework-agnostic core and provide component-based node views.

The document is **not HTML**. It is a ProseMirror document: a strict tree validated against the schema, JSON-serializable via `editor.getJSON()`. HTML is only an input/output format, converted through each node's `parseHTML`/`renderHTML` rules. Content that the schema does not describe is silently dropped on paste or `setContent`. This is the model's greatest strength (documents are always structurally valid and diffable) and its sharpest edge (you cannot "just insert some HTML" — you must model it as schema).

Collaboration is not built into the core. It is delivered through the `Collaboration` extension bound to a [Yjs](https://github.com/yjs/yjs) CRDT document, with the maintainers' [Hocuspocus](https://github.com/ueberdosis/hocuspocus) as the reference WebSocket backend. Conflict resolution is Yjs's, not Tiptap's.

## Production Notes

**The "multiple versions of prosemirror-model" error is the classic footgun.** ProseMirror uses `instanceof` checks internally, so two copies of `prosemirror-model` in the tree break silently or throw. `@tiptap/pm` exists specifically to force a single version; mixing it with directly-installed `prosemirror-*` packages, or having extensions that pull their own, reintroduces the problem. Deduping the lockfile is the fix.

**SSR needs care.** The editor touches the DOM on construction, so server rendering (Next.js App Router in particular) produces hydration mismatches unless you set `immediatelyRender: false` and let the editor mount client-side. This has been a recurring support topic.

**The ProseMirror abstraction leaks.** Custom node views, decorations, complex paste handling, and copy/serialization behavior all require dropping to ProseMirror APIs. Teams that budgeted for "a nice editor library" are frequently surprised by how much ProseMirror they end up learning. Read ProseMirror's guide before building custom nodes.

**Schema changes are migration hazards.** With collaboration, a schema mismatch between clients — or a schema change after documents already exist — can drop content or fail to load. Version your schema and plan content migrations; there is no automatic migration layer.

**Bundle size adds up.** Core plus a realistic set of extensions is not tiny; tree-shaking works but `StarterKit` pulls in everything it bundles. Import individual extensions when size matters.

**Major-version upgrades are real work.** The 1→2 jump was a full rewrite (see History); 2→3 changed package layout and some extension/option APIs and required following a migration guide. Pin versions and read the guide before bumping.

## When to Use / When Not

**Use when:**
- You want structured, schema-valid documents (JSON) rather than freeform HTML.
- You need a fully custom UI/design and are willing to build the toolbar yourself.
- You want real-time collaboration and are comfortable adopting Yjs/Hocuspocus.
- You're on React/Vue/Svelte and want a maintained binding rather than raw ProseMirror.

**Avoid when:**
- You want a batteries-included editor with a toolbar and dialogs out of the box (Tiptap ships none).
- You need to round-trip arbitrary HTML losslessly — the schema will drop what it doesn't model.
- Your team can't invest in learning ProseMirror for the non-trivial 20%.
- You need the Pro features (comments, versioning, AI, conversion) on a zero-budget project — those are paid.

## Alternatives

- ProseMirror/prosemirror — use directly when you want full control and no abstraction tax; you're already learning it anyway under Tiptap.
- facebook/lexical — use when you want a React-first, Meta-maintained editor with its own (non-ProseMirror) model and strong performance focus.
- ianstormtaylor/slate — use when you're React-only and want to control rendering entirely with your own components.
- slab/quill — use for simpler, drop-in editing needs where the Delta model and a default UI are acceptable.
- ckeditor/ckeditor5 — use when you want a full-featured, batteries-included editor and can accept its licensing/commercial model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019 | Initial release; Vue 2 only, thin ProseMirror wrapper[^1]. |
| 2.0 | 2023 | Full rewrite: framework-agnostic core, TypeScript, headless, chainable commands. Followed a long public beta[^4]. |
| 2.x | 2023–2025 | `@tiptap/pm` peer bundle, `immediatelyRender` SSR option, extension growth. |
| 3.0 | 2025 | Major release: package/API changes, migration guide required[^5]. |

## References

[^1]: Tiptap website and editor introduction. https://tiptap.dev/docs/editor/introduction
[^2]: ProseMirror — the underlying toolkit Tiptap is built on. https://prosemirror.net/
[^3]: Tiptap Pro extensions and subscription model. https://tiptap.dev/docs/editor/extensions
[^4]: Tiptap 2 announcement / documentation. https://tiptap.dev/
[^5]: Tiptap releases. https://github.com/ueberdosis/tiptap/releases

## Tags

typescript, rich-text-editor, wysiwyg, prosemirror, headless, editor, react, vue, contenteditable, collaboration, open-core
