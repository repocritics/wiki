# ProseMirror/prosemirror-view

> The display-and-input layer of the ProseMirror rich-text editor toolkit — the module that makes contentEditable behave.

[GitHub repo](https://github.com/ProseMirror/prosemirror-view) ·
[Official website](https://prosemirror.net) ·
[License: MIT](https://github.com/ProseMirror/prosemirror-view/blob/master/LICENSE)

## Overview

prosemirror-view is one of the four core modules of ProseMirror, Marijn Haverbeke's (also the CodeMirror author) toolkit for building rich-text editors on the web[^1]. The other three — prosemirror-model, prosemirror-state, prosemirror-transform — are pure data-structure libraries that could run in Node. This module is where the browser enters the picture: it renders an `EditorState` into a contentEditable DOM element, observes what the browser and the user do to that DOM, and converts those observations back into transactions.

The defining tension is that it embraces contentEditable rather than replacing it. Competing designs (Google Docs style) intercept all input and paint their own cursor and text; ProseMirror instead lets the browser handle cursoring, IME, spellcheck, and native text input, then reverse-engineers the browser's DOM mutations into clean document transactions. This buys native-feeling input and accessibility for free, at the cost of an ongoing war with browser inconsistencies — a large fraction of the module's code and commit history is workarounds for Chrome, Safari, Firefox, and Android IME behavior[^2].

Note on hosting: the GitHub repository was archived in 2026 and is now a read-only mirror; development moved to the author's self-hosted Forgejo instance at code.haverbeke.berlin, where issues and the changelog now live[^3]. The npm package (`prosemirror-view`) continues to be published and actively maintained (mirror last pushed 2026-04). GitHub star counts (~2,000) undercount usage: the module is a transitive dependency of Tiptap, Remirror, Milkdown, and in-house editors at Atlassian, GitLab, and The New York Times[^4].

## Getting Started

```bash
npm install prosemirror-view prosemirror-state prosemirror-model \
  prosemirror-schema-basic prosemirror-keymap prosemirror-commands
```

```js
import { EditorState } from "prosemirror-state"
import { EditorView } from "prosemirror-view"
import { schema } from "prosemirror-schema-basic"
import { keymap } from "prosemirror-keymap"
import { baseKeymap } from "prosemirror-commands"

const view = new EditorView(document.querySelector("#editor"), {
  state: EditorState.create({ schema, plugins: [keymap(baseKeymap)] }),
  dispatchTransaction(tr) {
    // Explicit unidirectional data flow — you own the state cycle.
    view.updateState(view.state.apply(tr))
  },
})
// When unmounting: view.destroy()
```

Nothing is bundled: without `baseKeymap`, Enter and Backspace at node boundaries do nothing. The `prosemirror-example-setup` package wires a usable default; production apps almost always assemble their own plugin stack (or adopt Tiptap, which does the assembly for you)[^5].

## Architecture / How It Works

`EditorView` holds a tree of **view descs** (`ViewDesc`), an internal shadow structure mapping each document node to the DOM node(s) that render it. On `updateState`, the view diffs the new document against this tree and patches only changed DOM subtrees — redraws are incremental, not virtual-DOM-style full re-renders. Rendering of individual nodes is delegated to the schema's `toDOM` specs via prosemirror-model's `DOMSerializer`, so the view module itself knows nothing about what a "paragraph" looks like.

Input flows the opposite way through a **DOM read-back loop**: a `MutationObserver` (plus composition and beforeinput events) detects that the browser mutated the contentEditable region, the view re-parses the affected range with the schema's `DOMParser`, diffs the parsed fragment against the current document, and synthesizes a transaction. This is the core trick — typing is not intercepted, it is observed and reconciled. Higher-level events (paste, drop, clicks, key events that map to commands) are intercepted before the browser acts, via plugin `props` such as `handleKeyDown`, `handlePaste`, and `handleDOMEvents`.

Two extension mechanisms matter most in practice:

- **Decorations** — widget (injected DOM), inline (styling ranges), and node decorations, held in a `DecorationSet`, a persistent tree keyed by document positions. Sets are *mapped* across transactions rather than rebuilt, which is what keeps per-keystroke cost low.
- **Node views** — per-node-type custom renderers that can take over a node's DOM entirely (e.g. embed a code editor or React component), optionally exposing a `contentDOM` hole where ProseMirror keeps managing editable children[^2].

Coupling is real but one-directional: prosemirror-view depends on model/state/transform and cannot be used without them, while those three never import the view. Versions across the four modules are released independently but expected to be kept reasonably current together.

## Production Notes

- **Android IME is the tax you pay.** Virtual keyboards (GBoard especially) compose text through mutation patterns that differ per keyboard and OS version. prosemirror-view handles most of it, but composition-related regressions are a recurring changelog theme; pin exact versions and retest text input on real Android devices — not emulators — after every upgrade.
- **Do not touch the editor DOM.** Anything inside the contentEditable that ProseMirror did not put there (browser extensions, Grammarly, stray framework re-renders) can be read back into the document or crash position mapping. React/Vue must never render *into* the editor element except via node views.
- **Decoration discipline.** Recomputing a large `DecorationSet` from scratch in a plugin's `apply` on every transaction is the most common self-inflicted performance bug; map the old set and patch only changed ranges. Thousands of widget decorations (e.g. comment markers in long documents) will still hurt — batch or virtualize.
- **Geometry APIs force layout.** `coordsAtPos` / `posAtCoords` trigger synchronous reflow; calling them in loops (custom selection overlays, tooltips tracking the cursor) causes jank in large documents.
- **Lifecycle leaks.** Forgetting `view.destroy()` in SPA unmounts leaks observers and event listeners. In React strict mode the double-mount will surface this immediately.
- **No SSR.** The view requires a live DOM; render document HTML server-side with `DOMSerializer` from prosemirror-model instead, and mount the view client-side.
- **Upgrade pain is low, assembly pain is high.** The 1.x API has been stable since 2017 — there has never been a v2 — but the flip side is a steep initial learning curve: schema, state, transactions, plugins, and decorations must all be understood before the first useful editor works. Budget weeks, not days, if you skip Tiptap.

## When to Use / When Not

**Use when:**
- You are building a custom rich-text or structured-document editor and need full control over schema, commands, and rendering.
- Collaborative editing is on the roadmap — the transaction model was designed for OT/CRDT integration (prosemirror-collab, y-prosemirror).
- You need documents to be a validated, schema-conforming data structure rather than sanitized HTML strings.

**Avoid when:**
- You need a drop-in editor this week — use Tiptap (same engine, batteries included) or Quill instead of raw ProseMirror modules.
- You are editing code, not prose — CodeMirror is the same author's purpose-built answer.
- Your "editor" is really a textarea with bold/italic; the schema and plugin machinery is heavy overkill for that.

## Alternatives

- ueberdosis/tiptap — same ProseMirror engine with a friendly extension API; use it when you want the power without assembling core modules.
- facebook/lexical — Meta's editor framework; use it when you prefer a maintained-by-a-large-org stack and React-first ergonomics.
- ianstormtaylor/slate — React-first editor framework; use it when your document model must be arbitrary JSON rather than a strict schema.
- quilljs/quill — batteries-included editor with Delta format; use it when you need a ready-made editor, not a framework.
- codemirror/dev — same author, for code editing; use it when the content is source code, not rich text.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-08 | ProseMirror announced + crowdfunding campaign; single-repo codebase[^1]. |
| 0.11 | 2016 | Monolith split into per-module packages; prosemirror-view born. |
| 1.0 | 2017-10 | Stable release across core modules; semver stability commitment[^5]. |
| 1.x | 2017–present | Continuous minor releases; no breaking major — API stable for 8+ years. |
| — | 2026 | Development moved to self-hosted forge (code.haverbeke.berlin); GitHub mirror archived[^3]. |

## References

[^1]: Marijn Haverbeke, "ProseMirror" announcement — 2015. https://marijnhaverbeke.nl/blog/prosemirror.html
[^2]: ProseMirror reference manual, view module. https://prosemirror.net/docs/ref/#view
[^3]: prosemirror-view README move notice; new home. https://code.haverbeke.berlin/prosemirror/prosemirror-view/
[^4]: ProseMirror project page (users, examples). https://prosemirror.net
[^5]: Marijn Haverbeke, "ProseMirror 1.0" — 2017-10-13. https://marijnhaverbeke.nl/blog/prosemirror-1.html

## Tags

typescript, rich-text-editor, contenteditable, wysiwyg, dom, browser, editor-framework, collaborative-editing, frontend, library
