# mermaid-js/mermaid

Diagrams as code — text-based flowcharts, sequence diagrams, gantt charts, mindmaps, and more, rendered inline in Markdown.

## What it is

A JavaScript library that takes a small text-DSL and renders it as an SVG diagram. GitHub, GitLab, Notion, Obsidian, and most Markdown environments render Mermaid blocks natively. Has become the de facto "diagrams in docs without leaving the editor" standard. Maintained by mermaid-chart Inc.

## Key features

- Diagram types: flowchart, sequence, gantt, class, state, ER, journey, gitgraph, mindmap, timeline, sankey, quadrant, requirement, C4, more.
- Text-DSL — version-controllable diagrams alongside code.
- SVG output for clean scaling.
- Native rendering in GitHub Markdown, GitLab, Notion, Obsidian.
- Live editor at mermaid.live.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- D3.js for layout primitives.
- Distributed via npm `mermaid` package.

## When to reach for it

- You're writing technical docs in Markdown and want diagrams alongside text.
- You want diagrams in version control as text rather than binary images.
- You're educating teams on architecture and want easy-to-update visuals.

## When *not* to reach for it

- You need precise, pixel-positioned diagrams — auto-layout is the trade-off Mermaid makes.
- You need diagrams beyond Mermaid's chart-type catalog.

## Maturity signal

88k stars, 9k forks, MIT, actively maintained. Commercial backing via Mermaid Chart funds ongoing dev. Open-issues count of 1,621 reflects the chart-type breadth.

## Alternatives

- PlantUML — older text-DSL for UML diagrams.
- D2 (Terrastruct) — newer text-DSL with better layout for some diagram types.
- Excalidraw — freehand drawing instead of text-DSL.
- draw.io / diagrams.net — GUI-based diagramming.

## Tags

typescript, diagrams-as-code, documentation, flowchart, mit-license, library, frontend, svg
