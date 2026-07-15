# facebook/lexical

> An extensible, framework-agnostic text editor framework from Meta — the successor to Draft.js, built around an immutable editor state on top of contentEditable.

[GitHub repo](https://github.com/facebook/lexical) ·
[Official website](https://lexical.dev) ·
[License: MIT](https://github.com/facebook/lexical/blob/main/LICENSE)

## Overview

Lexical is a text-editor framework maintained by Meta and open-sourced in 2022 as the replacement for the older, now-unmaintained Draft.js[^1]. Unlike most rich-text libraries, the core package (`lexical`) has no UI-framework dependency: it is plain TypeScript that reconciles an immutable editor state to a `contentEditable` DOM node. React is the first-class consumer via `@lexical/react`, but the same core drives non-React and headless usage. It powers production text input at Meta scale — Facebook, Messenger, WhatsApp Web, and Instagram comment surfaces[^1].

The defining tension is that Lexical is a *framework*, not a *drop-in editor*. There is no "toolbar with a bold button" out of the box. You assemble the editor from a core, a set of nodes, and plugins that register commands, listeners, and node transforms. This buys precise control, strong accessibility, and a serializable state model, at the cost of a real learning curve and a lot of wiring for anything beyond plain text. It is also, as of 2026, still on `0.x` semver — the API is production-hardened but explicitly not frozen, and minor releases carry breaking changes[^2].

With 23k+ stars, ~2.2k forks, and daily commit activity[^3], Lexical is one of the most actively developed editors in the JavaScript ecosystem. It sits opposite ProseMirror/TipTap and Slate in the current generation of "state-model-first" editors that replaced the contentEditable-as-source-of-truth designs of the 2010s.

## Getting Started

```bash
npm install lexical @lexical/react
```

```jsx
// A minimal rich-text editor with history (undo/redo).
import { LexicalComposer } from '@lexical/react/LexicalComposer';
import { RichTextPlugin } from '@lexical/react/LexicalRichTextPlugin';
import { ContentEditable } from '@lexical/react/LexicalContentEditable';
import { HistoryPlugin } from '@lexical/react/LexicalHistoryPlugin';
import { LexicalErrorBoundary } from '@lexical/react/LexicalErrorBoundary';
import { HeadingNode } from '@lexical/rich-text';

const initialConfig = {
  namespace: 'MyEditor',
  nodes: [HeadingNode],           // nodes must be registered up front
  onError: (error) => { throw error; },
};

export function Editor() {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <RichTextPlugin
        contentEditable={<ContentEditable />}
        placeholder={<div>Type here…</div>}
        ErrorBoundary={LexicalErrorBoundary}
      />
      <HistoryPlugin />
    </LexicalComposer>
  );
}
```

State is never mutated directly. All reads and writes go through scoped closures:

```jsx
import { $getRoot, $createParagraphNode, $createTextNode } from 'lexical';

editor.update(() => {
  const root = $getRoot();
  const p = $createParagraphNode();
  p.append($createTextNode('Hello'));
  root.append(p);
});
```

The `$`-prefixed functions (a Lexical convention, not a global) only work inside `editor.update()` or `editor.read()`. Calling them outside a closure throws — this is the single most common beginner error[^4].

## Architecture / How It Works

Lexical's core abstraction is an **immutable `EditorState`**: a tree of nodes (`RootNode`, `ElementNode`, `TextNode`, `DecoratorNode`, `LineBreakNode`, and user-defined subclasses) plus a selection. Each `editor.update()` produces a new, frozen `EditorState`; the editor keeps a *current* and a *pending* state (double-buffered), and a **reconciler** diffs them and applies the minimal set of DOM mutations to the `contentEditable` element. Because state is the source of truth and the DOM is a projection, undo/redo, collaboration, and serialization all operate on state rather than on parsed HTML.

The interaction model is **command-based**. Plugins call `editor.registerCommand(COMMAND, handler, priority)` and dispatch with `editor.dispatchCommand`. Commands propagate through priority tiers (low → critical) until a handler returns `true`, which lets plugins intercept and override each other deterministically. **Node transforms** (`editor.registerNodeTransform`) run during the update cycle to normalize the tree — e.g. merging adjacent text nodes or converting a typed `# ` into a heading — and are re-run to a fixpoint before the state is committed.

Packages are deliberately granular: `@lexical/rich-text`, `@lexical/plain-text`, `@lexical/list`, `@lexical/table`, `@lexical/code`, `@lexical/link`, `@lexical/markdown`, `@lexical/html`, `@lexical/selection`, `@lexical/utils`, and `@lexical/yjs`. `@lexical/react` wraps the core in a context (`LexicalComposer` + `useLexicalComposerContext`), and in that world a "plugin" is just a component that grabs the editor from context and registers listeners on mount. `@lexical/headless` runs the core without a DOM, for server-side state manipulation and tests.

Custom content is added by subclassing a node type and registering it in `initialConfig.nodes`. `DecoratorNode` is the escape hatch for embedding arbitrary UI (a React component, an image, a mention) into the editor tree while keeping it serializable. Every custom node must implement `importJSON`/`exportJSON` and carry a version, because the JSON serialization is the persistence contract.

The important honest point: Lexical does not replace `contentEditable`, it *governs* it. Browser input events (`beforeinput`, composition/IME events, clipboard) still originate from the native editable surface; Lexical intercepts and translates them into state updates. This is why it achieves good cross-browser behavior and accessibility, and also why edge cases in IME, paste, and selection still surface as real bugs.

## Production Notes

- **It is not an out-of-the-box editor.** A useful rich-text UI (toolbar, links, lists, tables, images, mentions, markdown shortcuts) is dozens of plugins and custom nodes. Teams routinely underestimate this; the [Playground](https://playground.lexical.dev) is the realistic reference for how much wiring a full editor takes.
- **`0.x` semver means real churn.** Minor releases have shipped breaking changes to node APIs, selection, and serialization. Pin exact versions and read release notes before upgrading; do not use caret ranges across a whole plugin set in production[^2].
- **Serialization is a migration liability.** Persisted documents are node JSON. When you change a custom node's shape or add a node type, old stored documents must still deserialize — version your nodes and write migrations, or you will fail to load historical content.
- **SSR needs `@lexical/headless` or client-only mounting.** The core touches the DOM; naive server rendering of the editable throws. Use the headless package to build/transform state on the server and hydrate on the client.
- **Selection is frequently `null`.** `$getSelection()` returns `null` when the editor is not focused, and returns different types (`RangeSelection`, `NodeSelection`, `TableSelection`). Handlers must type-guard every time, or you get intermittent runtime errors that only appear on blur/focus edge cases.
- **IME, paste, and mobile remain the hard edges.** Composition events (CJK input), rich paste from Word/Google Docs, and mobile browser quirks are where most non-trivial bugs live. Lexical handles the common cases well but you will still be reading `beforeinput` behavior across browsers for anything custom.
- **Collaboration via Yjs is capable but not turnkey.** `@lexical/yjs` gives real-time editing, but you own the provider, awareness/cursors, persistence, and conflict edge cases. Budget for it as its own subproject.
- **Bundle size scales with plugins.** The core is small and tree-shakeable, but a batteries-loaded editor (tables + code + markdown + collab) is meaningfully large. Code-split the editor away from initial page load.

## When to Use / When Not

**Use when:**
- You need a custom editing experience with full control over the document model and behavior, not a pre-built WYSIWYG.
- Accessibility and cross-browser reliability at scale matter, and you can invest in the framework.
- You want a serializable, immutable state model with clean undo/redo and optional real-time collaboration.
- You're in React (the best-supported binding) or need a framework-agnostic core.

**Avoid when:**
- You want a working toolbar-and-buttons editor with minimal code today — a batteries-included option gets you there faster.
- Your team can't absorb `0.x` breaking changes and the ongoing maintenance of custom nodes/plugins.
- You need a mature, frozen, long-term-stable API contract right now.
- Your use case is simple comment boxes where a lightweight editor or plain `<textarea>` with markdown suffices.

## Alternatives

- ProseMirror/prosemirror-view — schema-first, extremely mature and rigorous; use when you want a battle-tested model and don't mind assembling it yourself with even more low-level work.
- ueberdosis/tiptap — headless, batteries-included wrapper over ProseMirror; use when you want ProseMirror's power with far less setup and a large extension catalog.
- ianstormtaylor/slate — React-first, fully customizable editor; use when you're all-in on React and want deep control, accepting a less-frozen API.
- quilljs/quill — simpler, opinionated classic editor; use when you want a working rich-text box fast and don't need a custom document model.
- facebookarchive/draft-js — Lexical's predecessor; only relevant for legacy migrations — it is unmaintained and should not be chosen for new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-12-03 | Internal development begins at Meta[^3]. |
| public / v0.1 | 2022-04 | Open-sourced as the Draft.js successor[^1]. |
| 0.x series | 2022–2026 | Continuous `0.x` releases; framework-agnostic core, React bindings, Yjs collaboration, headless package, and an expanding node/plugin set. API not frozen[^2]. |

## References

[^1]: Meta Open Source / Lexical announcement and project background. https://lexical.dev/docs/intro
[^2]: Lexical releases and changelog (note the ongoing `0.x` versioning and breaking changes). https://github.com/facebook/lexical/releases
[^3]: GitHub repository metadata, facebook/lexical (created 2020-12-03, MIT, TypeScript; ~23.7k stars / ~2.2k forks as of 2026-07). https://github.com/facebook/lexical
[^4]: Lexical documentation, editor state and `$`-prefixed function conventions. https://lexical.dev/docs/concepts/editor-state

## Tags

typescript, javascript, text-editor, rich-text-editor, contenteditable, react, wysiwyg, editor-framework, meta, immutable-state, collaborative-editing
