# nhn/tui.editor

> A dual-mode Markdown editor: a GFM/CommonMark source editor with live preview and a ProseMirror-based WYSIWYG surface, switchable at runtime.

[GitHub repo](https://github.com/nhn/tui.editor) ·
[Official website](http://ui.toast.com/tui-editor) ·
[License: MIT](https://github.com/nhn/tui.editor/blob/master/LICENSE)

## Overview

TOAST UI Editor (`@toast-ui/editor`) is a browser Markdown editor built by NHN Cloud, the Korean cloud arm of NHN Corporation, and open-sourced in 2015. Its distinguishing feature is two coexisting editing modes over a single document: a **Markdown mode** (raw source with a synchronized rendered preview and scroll-sync) and a **WYSIWYG mode** (contenteditable-style rich editing), with a toolbar toggle to switch between them without losing content. It targets both CommonMark and GitHub Flavored Markdown, so tables, task lists, and strikethrough round-trip through both modes.[^1]

The current line is a ground-up rewrite. Versions 1.x and 2.x were jQuery- and CodeMirror-based; version 3.0 (2021) discarded that stack, moved the whole codebase to TypeScript, replaced the WYSIWYG engine with **ProseMirror**, and replaced the Markdown parser with **ToastMark**, an in-house CommonMark parser that tracks source positions for incremental live preview.[^2] The rewrite is why the 3.x API is not a drop-in upgrade from 2.x and why most third-party tutorials predating 2021 are wrong.

The defining tension today is maintenance. The repo is popular (18k+ stars) and still the most feature-complete open-source Markdown-WYSIWYG combo, but the last commit to `master` was in August 2024 and the issue tracker carries 600+ open issues.[^3] It is best treated as a stable, feature-frozen component rather than an actively evolving one — a real consideration given ProseMirror underneath continues to move.

## Getting Started

```bash
npm install @toast-ui/editor
```

```js
import '@toast-ui/editor/dist/toastui-editor.css';
import { Editor } from '@toast-ui/editor';

const editor = new Editor({
  el: document.querySelector('#editor'),
  height: '500px',
  initialEditType: 'markdown',   // or 'wysiwyg'
  previewStyle: 'vertical',      // side-by-side live preview
  initialValue: '# Hello\n\n- [x] task list item',
});

// Markdown out, regardless of which mode the user last touched
console.log(editor.getMarkdown());
```

React and Vue wrappers ship as separate packages (`@toast-ui/react-editor`, `@toast-ui/vue-editor`). Note the Vue wrapper targets Vue 2; Vue 3 support was a long-standing open request that never landed officially.[^4]

## Architecture / How It Works

The editor is not one editor but two rendering engines coordinated over a shared Markdown source of truth:

1. **Markdown mode** — a source editor that parses input with **ToastMark**, NHN's fork/reimplementation of the CommonMark reference parser. ToastMark's purpose is incremental parsing: on each keystroke it re-parses only the affected node range and keeps an AST with source offsets, which is what makes the live HTML preview and syntax highlighting update without re-rendering the whole document.[^2]
2. **WYSIWYG mode** — a **ProseMirror** document. ProseMirror enforces a schema (a document is a validated tree of known node/mark types, not free-form `contenteditable` HTML), which is what gives the WYSIWYG mode predictable table editing, custom blocks, and paste normalization from Word/Excel/screenshots.

Switching modes serializes one representation to Markdown and re-parses into the other, so Markdown is the interchange format between the two engines. Anything a plugin or custom block introduces must define how it converts in both directions or it will not survive a mode switch.

**Plugins** are the extension surface. Five are maintained in-tree — `chart` (renders fenced ```chart``` blocks via TOAST UI Chart), `code-syntax-highlight` (Prism.js), `color-syntax` (color picker), `table-merged-cell` (rowspan/colspan in tables, a GFM extension), and `uml` (PlantUML rendering). A plugin registers Markdown parsing rules, ProseMirror schema nodes, toolbar items, and renderers as needed; the `chart`/`uml` plugins are the canonical examples of custom fenced-block handling.

The build is an npm workspaces monorepo (`apps/editor`, the framework wrappers, and `plugins/*`) requiring npm 7+, with snowpack for the dev server and webpack retained for legacy-browser builds.[^1]

## Production Notes

**CSS is not optional and not scoped.** You must import `toastui-editor.css` (and each plugin's CSS) yourself; forgetting it yields an unstyled, broken-looking editor. The styles are global class names (`.toastui-editor-*`), so they can collide with or be overridden by app CSS, and there is no built-in CSS-module/shadow-DOM isolation.

**Bundle size.** Pulling in the full editor plus chart/uml/syntax-highlight plugins is heavy — Prism.js and TOAST UI Chart are substantial dependencies. If you only need the read-only render path, ship the **Viewer** build (`Editor.factory({ viewer: true })` or the `@toast-ui/editor/viewer` entry) instead of the full editor; it omits the editing machinery.

**Maintenance risk is the headline caveat.** With `master` untouched since mid-2024 and hundreds of open issues, do not plan around upstream fixes. Known long-tail gaps include Vue 3, some GFM edge cases, and ProseMirror version drift. Teams standardizing on it should be prepared to vendor patches or fork.

**IE11 baggage.** The project still advertises IE11 support, which constrained its build and API choices. On modern-only targets this is dead weight but generally harmless; it mostly shows up as the extra webpack legacy build path.

**Markdown fidelity across modes.** Because mode switching serializes through Markdown, constructs Markdown cannot express (arbitrary inline HTML, certain nested structures, plugin nodes without a serializer) can be lost or normalized when a user toggles WYSIWYG↔Markdown. Test the specific content shapes your users produce, not just the happy path.

**Framework wrappers are thin.** The React/Vue components wrap the imperative instance; you reach the real API through a ref (`editorRef.current.getInstance()`). Treating them as fully declarative React/Vue components leads to fighting the wrapper.

## When to Use / When Not

**Use when:**
- You want a drop-in editor that gives end users *both* a raw-Markdown mode and a rich WYSIWYG mode over the same document, out of the box.
- You need GFM tables, task lists, and paste-from-Office/screenshot handling without assembling ProseMirror yourself.
- You need built-in i18n (20+ languages), a dark theme, and a viewer/read-only mode without extra work.
- You accept a feature-frozen but stable dependency.

**Avoid when:**
- You need an actively maintained editor with responsive upstream fixes — the repo is effectively dormant.
- You want a headless, un-styled editor you fully control (build on ProseMirror or TipTap instead).
- You're on Vue 3 and want a first-class supported wrapper.
- You only need plain Markdown source editing (a source-only editor like CodeMirror is far lighter).

## Alternatives

- ueberdosis/tiptap — headless ProseMirror toolkit, actively maintained; use when you want to build your own UI and need ongoing upstream support.
- ProseMirror/prosemirror — the low-level engine tui.editor's WYSIWYG is built on; use when you need full control over schema and behavior.
- codemirror/dev — source/code editor (CodeMirror 6); use when you want plain Markdown editing with no WYSIWYG mode and minimal weight.
- ianstormtaylor/slate — React-first customizable editor framework; use when your stack is React and you want to model documents yourself.
- codex-team/editor.js — block-style editor emitting JSON; use when you want structured block output rather than Markdown text.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial open-sourcing by NHN; jQuery + CodeMirror + Squire era. |
| 1.0 | 2018 | First stable line; CodeMirror Markdown mode, contenteditable WYSIWYG. |
| 2.0 | 2020 | Dropped jQuery dependency; API cleanup. |
| 3.0 | 2021-06 | Full rewrite: TypeScript, ProseMirror WYSIWYG, ToastMark parser.[^2] |
| 3.2.x | 2022–2024 | Incremental plugin/bugfix releases; last active development.[^3] |

## References

[^1]: TOAST UI Editor README and package layout, nhn/tui.editor. https://github.com/nhn/tui.editor
[^2]: ToastMark parser and the 3.0 rewrite, nhn/tui.editor `apps/editor` / `libs/toastmark`. https://github.com/nhn/tui.editor/tree/master/libs/toastmark
[^3]: Repository activity — last push 2024-08-01, 600+ open issues (GitHub API, retrieved 2026-07). https://github.com/nhn/tui.editor/issues
[^4]: Vue 3 support request thread, nhn/tui.editor issues. https://github.com/nhn/tui.editor/issues

## Tags

markdown, wysiwyg-editor, prosemirror, commonmark, gfm, typescript, rich-text-editor, javascript, react, vue, browser, frontend
