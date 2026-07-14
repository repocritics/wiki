# xyflow/xyflow

> Node-based UI libraries for React and Svelte — the de facto default for building flow editors, diagrams, and visual workflow builders on the web.

[GitHub repo](https://github.com/xyflow/xyflow) ·
[Official website](https://xyflow.com) ·
[License: MIT](https://github.com/xyflow/xyflow/blob/main/LICENSE)

## Overview

xyflow is a monorepo maintained by the xyflow team (formerly webkid, a Berlin data-visualization studio)[^1]. It is the home of React Flow, the library that popularized the node-and-edge editor pattern in React, and its younger sibling Svelte Flow. The repository was originally `wbkd/react-flow`; it was renamed and restructured into the multi-package `xyflow/xyflow` monorepo in 2023 when the team extracted a framework-agnostic core and added Svelte support[^2]. With ~37k stars it is the dominant choice for anyone building a whiteboard, flowchart, no-code workflow builder, or graph editor in the React ecosystem.

The defining design decision is that nodes are rendered as ordinary DOM elements (React or Svelte components), not drawn on a canvas or WebGL surface. This is why React Flow feels natural — a node is just a component, you style it with CSS and wire up event handlers as usual — and it is also the library's central constraint: DOM-based rendering is flexible but does not scale to tens of thousands of nodes the way a canvas renderer would. Everything about performance tuning in production traces back to this tradeoff.

The libraries are MIT-licensed with no feature gating. Revenue comes from React Flow Pro — a subscription that funds development in exchange for pro examples, priority issue handling, and support, not a separate proprietary build[^3].

## Getting Started

```sh
npm install @xyflow/react
```

```jsx
import { useCallback } from 'react';
import {
  ReactFlow, MiniMap, Controls, Background,
  useNodesState, useEdgesState, addEdge,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';   // required — omit this and nothing renders

const initialNodes = [
  { id: '1', position: { x: 0, y: 0 }, data: { label: 'Node 1' } },
  { id: '2', position: { x: 0, y: 100 }, data: { label: 'Node 2' } },
];
const initialEdges = [{ id: 'e1-2', source: '1', target: '2' }];

export default function Flow() {
  const [nodes, , onNodesChange] = useNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialEdges);
  const onConnect = useCallback(
    (params) => setEdges((eds) => addEdge(params, eds)), [setEdges]);

  // The parent MUST have an explicit width/height or the canvas is 0px tall.
  return (
    <div style={{ width: '100%', height: '100vh' }}>
      <ReactFlow
        nodes={nodes} edges={edges}
        onNodesChange={onNodesChange} onEdgesChange={onEdgesChange}
        onConnect={onConnect} fitView
      >
        <MiniMap /><Controls /><Background />
      </ReactFlow>
    </div>
  );
}
```

`@xyflow/svelte` mirrors this API for Svelte.

## Architecture / How It Works

The monorepo ships four packages:

1. **`@xyflow/system`** — a framework-agnostic core in vanilla TypeScript. It owns the hard parts: pan/zoom (`XYPanZoom`), node dragging (`XYDrag`), connection/handle logic (`XYHandle`), and edge-path geometry. It builds on d3-zoom / d3-drag / d3-selection for the low-level pointer and transform math. React Flow and Svelte Flow are thin, idiomatic bindings over this shared engine — which is how one team keeps two frameworks in feature parity.
2. **`@xyflow/react`** — React Flow 12, the current line.
3. **`reactflow`** — React Flow 11, the previous single-package release, maintained on the `v11` branch.
4. **`@xyflow/svelte`** — Svelte Flow.

Internally React Flow holds all interactive state (node positions, selection, viewport transform, in-progress connections) in a Zustand store. The viewport is a single CSS-transformed `<div>`; each node is absolutely positioned inside it and edges are rendered as SVG `<path>` elements in an overlay. Zooming and panning change one CSS transform rather than re-laying-out every node, which keeps interaction smooth. Custom nodes are your own components registered through `nodeTypes`; custom edges through `edgeTypes`.

The library is controlled by default: `nodes` and `edges` are your state, and you apply the library's proposed mutations yourself via `applyNodeChanges` / `applyEdgeChanges` (the `useNodesState`/`useEdgesState` hooks are convenience wrappers that do this for you). React Flow computes rendered node dimensions after mount — in v12 measured sizes live on `node.measured.{width,height}` — which is what makes edges connect to the true edges of arbitrarily-sized custom nodes.

## Production Notes

**The two most common footguns are structural.** First, the flow renders into its parent's box, so the parent needs an explicit height; a `height: 0` container renders an invisible canvas and generates a stream of "did you forget to add dimensions" support questions. Second, `nodeTypes` and `edgeTypes` must be stable references — defining them inline inside the component body creates a new object every render, forces React Flow to re-register node types, and tanks performance. Define them at module scope or wrap in `useMemo`; the library warns about this in the console but the warning is easy to miss.

**Scaling is DOM-bound.** Because every node is a live DOM subtree, render cost grows with node/edge count. Hundreds of nodes are fine; low thousands need care; tens of thousands are outside the library's comfort zone. Mitigations: `onlyRenderVisibleElements` to cull off-screen nodes, memoizing custom node components, keeping node `data` shallow, and avoiding expensive work inside node render. For genuinely large graphs (network/graph-theory visualization), a canvas-based library is the better fit — see Alternatives.

**Upgrading v11 → v12 is a real migration, not a version bump.** The package name changed from `reactflow` to `@xyflow/react`, the CSS import path changed, node dimension fields moved onto `measured`, and several store internals were renamed. Plan for a coordinated sweep across imports, types, and CSS rather than an in-place bump[^2]. v12 also added server-side rendering support (nodes render with provided width/height hints before hydration), which was not possible in v11.

**Controlled state discipline.** Selection, dragging, and connecting all emit change events that you must apply back to state — skip that and nodes appear frozen. Teams new to the controlled model frequently file "dragging doesn't work" issues that resolve to a missing `onNodesChange`. Layout is not included: React Flow positions nodes at the coordinates you give it. Automatic layout is delegated to external libraries (dagre, elkjs, d3-hierarchy) wired into your own effect.

## When to Use / When Not

**Use when:**
- You're building a flow editor, diagram tool, no-code/automation builder, or pipeline/DAG UI in React or Svelte.
- You want nodes to be real, styleable, interactive components rather than canvas draw calls.
- You need built-in pan/zoom, minimap, controls, connection handles, and selection without assembling them yourself.

**Avoid when:**
- You need to render tens of thousands of nodes or run graph-theory layouts at scale — DOM rendering will be the bottleneck.
- You want a freeform drawing/whiteboard canvas rather than a structured node-edge graph.
- You're not on React or Svelte — there is no first-party binding for other frameworks (though `@xyflow/system` is framework-agnostic if you want to build one).

## Alternatives

- tldraw/tldraw — use instead when you want a freeform infinite whiteboard/drawing canvas rather than a structured node-and-edge graph.
- excalidraw/excalidraw — use instead for hand-drawn-style diagramming and collaborative sketching, not programmatic flow editors.
- cytoscape/cytoscape.js — use instead for large-scale network/graph-theory visualization with canvas rendering and built-in graph algorithms.
- retejs/rete — use instead when you want a node editor centered on dataflow/logic graphs with a processing engine, framework-agnostic.
- projectstorm/react-diagrams — use instead if you specifically want an older, SVG-based React diagramming library; smaller community than React Flow today.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-07 | `wbkd/react-flow` created by the webkid team[^1]. |
| 10.x | 2022 | Maturing React Flow API, growing adoption. |
| 11.x | 2022-11 | `reactflow` single-package release; the long-lived v11 line[^4]. |
| monorepo | 2023 | Repo renamed to `xyflow/xyflow`; `@xyflow/system` core extracted, Svelte Flow added[^2]. |
| 12.x | 2024 | `@xyflow/react`; SSR support, `measured` dimensions, package rename from `reactflow`[^2]. |

## References

[^1]: xyflow team / about — the maintainers, formerly webkid. https://xyflow.com/about
[^2]: xyflow monorepo README — package layout (`@xyflow/react`, `reactflow` v11, `@xyflow/svelte`, `@xyflow/system`) and migration context. https://github.com/xyflow/xyflow
[^3]: React Flow Pro — subscription funding model; libraries remain MIT. https://reactflow.dev/pro
[^4]: React Flow changelog / releases. https://github.com/xyflow/xyflow/releases

## Tags

typescript, react, svelte, node-based-ui, flowchart, diagram, workflow-builder, graph, dataviz, frontend-library, monorepo
