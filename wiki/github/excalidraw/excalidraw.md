# excalidraw/excalidraw

A virtual whiteboard for sketching hand-drawn-style diagrams — the canonical "let me draw a quick architecture diagram" tool for software teams.

## What it is

A web-based diagramming canvas where every shape is rendered in a deliberately rough, hand-drawn style. The aesthetic is the point: the rough-strokes look signals "this is a sketch, don't take it as final". Used for system-design discussions, architecture diagrams in docs, whiteboard-style code reviews, and informal collaboration. Local-first by default — sessions persist in the browser; an optional realtime-collaboration backend is also provided. Hostable at excalidraw.com or self-hosted from this repo.

## Key features

- Hand-drawn-style rendering for shapes, arrows, and text — the signature aesthetic.
- Real-time collaboration over WebRTC (when the optional backend is configured).
- Library / library-shape system for reusable component primitives.
- `.excalidraw` file format — JSON-encoded canvas state, fully portable.
- Embeddable as a React component (`@excalidraw/excalidraw` npm package).
- Excalidraw+ commercial offering for hosted collaboration; this repo is the open-source core.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- HTML5 Canvas as the renderer.
- React component model — the editor is also packaged for embedding into other apps.
- Backend (optional collab service) is a separate Express + WebSocket server in a sister repo.

## When to reach for it

- You need a quick architecture diagram for a doc, blog post, or RFC.
- You're whiteboarding with a remote team and want a tool that's faster to start than Miro.
- You're embedding diagramming into your own app — the React package handles the editor in-line.

## When *not* to reach for it

- You need precise, schematic-style diagrams — Mermaid or Lucidchart are closer-fit.
- You need long-running, multi-tenant collaboration with permissions — Miro / FigJam handle that, Excalidraw is whiteboard-first.
- You want auto-layout — Excalidraw is freehand-positioning; nothing is computed for you.

## Maturity signal

124k stars, 14k forks, MIT, last push 2026-06-01 — actively maintained. 6-year-old project that's become the de facto "sketch a diagram quickly" tool in dev culture. Open-issues count of 3,123 reflects the breadth of feature requests; the team's focus is curated rather than chasing the issue tracker. The Excalidraw+ commercial product funds the OSS work.

## Alternatives

- Mermaid — use when you want diagrams generated from text rather than drawn freehand.
- Miro / FigJam — use for multi-team workspaces with permissions and persistent boards.
- tldraw — direct competitor; tldraw is more polished as an SDK, Excalidraw is more entrenched as a destination.
- diagrams.net (formerly draw.io) — use for technical/precise diagrams.

## Notes

The hand-drawn aesthetic is intentional and load-bearing — it's why Excalidraw spread organically among engineers. The file format is plain JSON, so diagrams round-trip cleanly through version control. License (MIT) plus the embed-as-React-component path makes this the easiest diagramming tool to vendor into a custom app.

## Tags

typescript, react, canvas, whiteboard, diagrams, drawing, productivity, collaboration, user-interface
