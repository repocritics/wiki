# ianstormtaylor/slate

> A schema-less, plugin-first framework for building rich text editors on top of contenteditable and React — still 0.x after nearly a decade.

[GitHub repo](https://github.com/ianstormtaylor/slate) ·
[Official website](http://slatejs.org) ·
[License: MIT](https://github.com/ianstormtaylor/slate/blob/main/License.md)

## Overview

Slate is a toolkit for building rich text editors — the kind of surfaces behind Medium, Dropbox Paper, or Google Docs — rather than a drop-in editor component. It gives you a document model, a set of transform primitives, and a React rendering layer, and expects you to assemble the actual editor from plugins and your own components. It was created by Ian Storm Taylor in 2016 and was explicitly inspired by Draft.js, ProseMirror, and Quill, positioning itself as the "schema-less, do-it-yourself" option among them.

The defining tension is stated in the README itself: Slate is still labelled beta with no 1.0 schedule, despite being one of the most-starred editor frameworks on GitHub and shipping in production widely[^1]. It is contributor-driven with no corporate backing, so advanced use cases and bug fixes often depend on you sending the pull request. Teams adopt it for the flexibility — a nested, DOM-parallel document tree with no baked-in schema — and accept an API that can still change across 0.x releases.

One structural fact to know up front: today's Slate is a complete rewrite (the 0.50 line, late 2019) that replaced the original Immutable.js-based model with plain JSON objects and a TypeScript codebase[^2]. Much of the tutorial content and StackOverflow history online predates that rewrite and no longer applies.

## Getting Started

```bash
npm install slate slate-react react react-dom
```

```tsx
import { useState } from 'react'
import { createEditor, Descendant } from 'slate'
import { Slate, Editable, withReact } from 'slate-react'

const initialValue: Descendant[] = [
  { type: 'paragraph', children: [{ text: 'A line of text in a paragraph.' }] },
]

export default function App() {
  const [editor] = useState(() => withReact(createEditor()))
  return (
    <Slate editor={editor} initialValue={initialValue}>
      <Editable placeholder="Start typing…" />
    </Slate>
  )
}
```

The document is plain JSON: an array of `Element` and `Text` nodes. There is no schema definition step — `type: 'paragraph'` is your convention, not Slate's. You render nodes with your own `renderElement` / `renderLeaf` callbacks.

## Architecture / How It Works

Slate splits into a view-agnostic core and a React binding, published as separate packages from a Lerna monorepo:

- **`slate`** — the data model and logic. Defines the document as a recursive tree of `Element` and `Text` nodes, plus the interfaces `Editor`, `Node`, `Path`, `Point`, `Range`, `Operation`, and the `Transforms` helpers. The tree is immutable-by-convention (Immer under the hood), not enforced by a class.
- **`slate-react`** — renders the model into a `contenteditable` `<div>` and translates DOM events (typing, paste, selection, composition) back into Slate operations. This is where the hard browser work lives.
- **`slate-history`** — undo/redo, layered on as a plugin.
- **`slate-hyperscript`** — JSX authoring of Slate documents, mostly for tests.

Two ideas hold the design together. First, **every change is an operation:** editing produces low-level operations (`insert_text`, `remove_node`, `set_selection`, …) applied to the editor. Because all mutation flows through operations, undo history and collaborative editing can be layered on without rewriting the model — the payoff behind the "collaboration-ready" claim, though collaboration itself is not included.

Second, **plugins are just functions.** A plugin takes an editor and returns one, typically overriding methods: `withReact`, `withHistory`, or your own wrapper overriding `editor.normalizeNode`, `editor.isInline`, `editor.isVoid`, or `editor.insertData`. There is no plugin array or registry — composition is `withMyPlugin(withReact(createEditor()))`.

Normalization is the closest thing to a schema: `normalizeNode` runs after each change to enforce invariants (e.g. blocks must contain text). It is powerful and also the most common source of bugs — a normalizer that never reaches a stable state produces an infinite loop.

## Production Notes

**contenteditable is the substrate, with all that implies.** Slate abstracts many browser inconsistencies but cannot hide them all. Selection handling, copy/paste fidelity, and native spellcheck behavior still leak through.

**Input Method Editors and Android are the weakest area.** Composition input (CJK IMEs) and Android soft keyboards have historically been the most-reported class of Slate bugs; `slate-react` carries dedicated input-handling code for these paths, but mobile and IME editing remain the first thing to test on a real device, not an emulator.

**Beta means pin your versions.** Breaking changes ship in 0.x minor releases; use exact versions and read release notes before bumping. Community packages (`slate-yjs`, table/embed helpers, serializers) track core loosely, so a core bump can break them.

**Large documents degrade.** There is no built-in virtualization; thousands of nodes, or decorations that recompute across the whole document (e.g. live syntax highlighting), can stall the editor. Budget for chunking or custom windowing.

**Collaboration is not included.** The operation model makes CRDT collaboration feasible and `slate-yjs` (community) is the usual route, but the Yjs-to-Slate normalization binding has sharp edges you own.

**SSR / React frameworks.** `Editable` must run on the client; under Next.js App Router it needs `'use client'`, and contenteditable/selection hydration mismatches are common — render after mount if you hit them.

## When to Use / When Not

**Use when:**
- You need a genuinely custom editing surface (domain-specific blocks, embeds, nested structures) and want no schema assumptions fighting you.
- You are already in React and want the view layer to be your own components.
- You want a plain-JSON document you can serialize and reason about directly.

**Avoid when:**
- You want a batteries-included editor with a stable 1.0 and vendor support — Slate is beta and contributor-driven.
- Your primary target is mobile / heavy IME input and you cannot absorb contenteditable edge cases.
- You need a rigid, enforced schema and built-in, battle-tested collaboration out of the box — ProseMirror is stricter here.

## Alternatives

- facebook/lexical — Meta's newer, framework-agnostic editor and Draft.js successor; use it when you want an actively maintained editor with strong performance and a stable release line.
- ProseMirror/prosemirror-view — schema-enforced model with mature collaborative editing; use it when correctness and a strict document schema matter more than flexibility.
- ueberdosis/tiptap — headless wrapper over ProseMirror with a friendlier API; use it when you want ProseMirror's rigor without wiring it by hand.
- quilljs/quill — Delta-based and more turnkey; use it when you want a working editor fast and don't need a fully custom schema.
- facebookarchive/draft-js — the React editor Slate reacted against; now archived, so treat it as legacy only.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Initial release. Immutable.js-based model, React view[^1]. |
| 0.47 | 2019 | Last major release of the original architecture. |
| 0.50 | 2019-11 | Full rewrite: plain-JSON model, TypeScript, Immer, function-composition plugins[^2]. |
| 0.6x+ | 2021+ | `initialValue` prop and incremental refinements; still beta, no 1.0. |

## References

[^1]: Slate README and project site — history, principles, and beta status. https://github.com/ianstormtaylor/slate and http://slatejs.org
[^2]: Slate documentation, "Concepts" — the 0.50 rewrite to a plain-JSON, TypeScript core. http://docs.slatejs.org/concepts

## Tags

typescript, react, rich-text-editor, contenteditable, wysiwyg, editor-framework, javascript, plugin-architecture, frontend, text-editor
