# cytoscape/cytoscape.js

> Graph theory (network) library for visualisation and analysis — a headless graph model plus an optional interactive renderer.

[GitHub repo](https://github.com/cytoscape/cytoscape.js) ·
[Official website](https://js.cytoscape.org) ·
[License: MIT](https://github.com/cytoscape/cytoscape.js/blob/unstable/LICENSE)

## Overview

Cytoscape.js is a JavaScript library for representing and drawing graphs (nodes connected by edges). It began at the University of Toronto's Bader Lab as the browser sibling of the desktop Cytoscape network-analysis tool, and was published in Oxford Bioinformatics in 2016 with an update paper in 2023[^1][^2]. Its home turf is bioinformatics and network biology, but it is used widely for anything relational: social graphs, dependency maps, provenance/lineage views, knowledge graphs, and workflow editors.

The defining design choice is the split between a **graph model** and a **renderer**. The core is a headless, in-memory graph data structure with a jQuery-like traversal and mutation API plus a library of graph-theory algorithms. Rendering is optional: pass a `container` DOM element and you get an interactive HTML5 Canvas view; omit it and the same code runs server-side under Node.js for pure analysis. This separation is why the library works both as a visualization widget and as an analysis engine.

The tradeoff is that Cytoscape.js is a graph library, not a general drawing library. Its styling and layout vocabulary is expressive for node/edge diagrams and deliberately narrow for anything else. It historically rendered only to 2D Canvas (no SVG, no built-in WebGL), which sets a practical ceiling on how many elements you can display interactively — the recurring production concern discussed below.

## Getting Started

```bash
npm install cytoscape
```

```js
import cytoscape from 'cytoscape';

const cy = cytoscape({
  container: document.getElementById('cy'),
  elements: [
    { data: { id: 'a' } },
    { data: { id: 'b' } },
    { data: { id: 'ab', source: 'a', target: 'b' } },
  ],
  style: [
    { selector: 'node', style: { 'background-color': '#666', label: 'data(id)' } },
    { selector: 'edge', style: { width: 2, 'line-color': '#ccc' } },
  ],
  layout: { name: 'grid' },
});

// Same API works headless (no container) for analysis:
const bfs = cy.elements().bfs({ root: '#a' });
console.log(bfs.path.map(el => el.id()));
```

## Architecture / How It Works

Four subsystems make up the library:

1. **Graph model** — an in-memory collection of elements (nodes and edges), each carrying a `data` object plus internal state (position, selection, classes). Elements are addressed through a **selector engine** modeled on CSS/jQuery: `cy.$('node[weight > 50]')` returns a collection you can traverse (`.neighborhood()`, `.connectedEdges()`), mutate, or animate. Collections are immutable views over the live graph.

2. **Stylesheet** — appearance is defined by CSS-like rules matched against selectors, not set imperatively per element. Style properties can be constants or **data-driven mappers** (`data(...)`, `mapData(...)`), so visual encoding follows the underlying data. This is powerful but means a full restyle re-evaluates rules across matched elements.

3. **Layout** — algorithms that assign node positions. Built-ins include `grid`, `circle`, `concentric`, `breadthfirst`, `preset` (use supplied coordinates), and `cose`, a force-directed layout. Higher-quality or specialized layouts (`fcose`, `cola`, `dagre`, `elk`, `klay`, `cise`, `euler`) ship as separate extensions rather than core.

4. **Renderer** — the default renderer paints to a 2D Canvas and handles hit-testing, panning, zooming, and gestures. The renderer is pluggable in principle; in practice Canvas is what nearly everyone uses.

The **extension system** is central to how the project scales feature-wise. Layouts, UI widgets (edge handles, context menus, popovers), and exporters live as ~70 first- and third-party extensions[^3] rather than in core, which keeps the base bundle focused. `cytoscape.use(ext)` registers one. This keeps core small at the cost of an ecosystem whose quality and maintenance vary per extension.

## Production Notes

**Element count is the ceiling.** The Canvas renderer redraws interactively, and cost scales with the number of visible nodes, edges, and — especially — labels and complex edge styles. A few thousand elements is comfortable; tens of thousands strains interactivity. Mitigations that actually move the needle: batch mutations with `cy.batch(() => { ... })` to collapse redraws, enable `hideEdgesOnViewport` and `textureOnViewport` during pan/zoom, disable `motionBlur` if it hurts, and avoid expensive edge styles (`haystack` edges and hidden labels are much cheaper than bezier edges with rich labels). A recent experimental WebGL renderer has been introduced to raise this ceiling, but treat it as opt-in and verify behavior for your styles before relying on it[^4].

**Layout is often the real bottleneck.** Force-directed layouts (`cose`, `fcose`, `cola`) can dominate frame time on large graphs and block the main thread. Prefer worker-capable layouts (`cola`, `elk`) for big graphs, run layout once and cache positions (`preset`) where possible, and constrain iterations. Layout choice matters more for perceived performance than renderer tuning.

**Bulk updates need batching.** Adding elements or changing data/style one call at a time triggers style recalculation and redraw each time. Wrap bulk changes in `cy.batch()` or `cy.startBatch()`/`cy.endBatch()`; forgetting this is the most common cause of "why is loading 5,000 nodes so slow."

**Framework integration is via wrappers.** Cytoscape.js manages its own DOM/Canvas imperatively, which does not fit React's declarative model cleanly. The community `react-cytoscapejs` wrapper and Angular/Vue equivalents exist, but you still reason in terms of a `cy` instance and imperative mutations; state-sync between framework state and the graph is your responsibility.

**Memory and teardown.** Long-lived single-page apps that create and discard graphs should call `cy.destroy()` to release listeners and Canvas resources; leaked instances and event handlers are a real source of growth in dashboards.

## When to Use / When Not

**Use when:**
- You need both a graph data model and interactive visualization from one library.
- You want CSS-like declarative styling and data-driven visual encoding of a network.
- You need graph algorithms (traversal, shortest paths, centrality, components) alongside rendering — or headless in Node.js.
- Your graphs are in the hundreds-to-low-thousands of elements and interactivity matters.

**Avoid when:**
- You need to render very large graphs (tens of thousands+ of elements) at interactive frame rates — a WebGL-first library fits better.
- You want fully bespoke, non-network visualizations — a lower-level primitive like D3 gives more control.
- You only need static, non-interactive diagrams — a layout-to-SVG tool (Graphviz, Mermaid) is simpler.
- You need only graph algorithms with no rendering — a pure data/algorithm library is lighter.

## Alternatives

- sigmajs/sigma — WebGL graph renderer that scales to far larger graphs; pair with graphology for the data model. Use instead when element count is the constraint.
- jacomyal/graphology — pure graph data structure and algorithms, no rendering. Use when you want analysis only and will render (if at all) separately.
- visjs/vis-network — simpler, more turnkey network widget. Use when you want quick results and less configuration over extensibility.
- antvis/G6 — feature-rich graph visualization with Canvas/WebGL. Use when you want a batteries-included alternative, especially in that ecosystem.
- d3/d3 — low-level building blocks, not a graph library. Use when you need full control over a custom visualization and are willing to build the graph layer yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011-08 | Repository created at University of Toronto (Bader Lab)[^1]. |
| 2.0 | 2014 | Established the model + Canvas renderer + stylesheet architecture. |
| — | 2016 | Oxford Bioinformatics paper (Franz et al.)[^1]. |
| 3.0 | 2017 | Major line; ES modules, extension API maturation. Long-lived 3.x series. |
| — | 2023 | Bioinformatics update paper[^2]. |
| 3.x | ongoing | Monthly feature releases, weekly patch releases; development on the `unstable` branch, stable on `master`[^5]. |

## References

[^1]: Franz M, Lopes CT, Huck G, Dong Y, Sumer O, Bader GD. "Cytoscape.js: a graph theory library for visualisation and analysis." Bioinformatics 32(2):309–311, 2016. https://academic.oup.com/bioinformatics/article/32/2/309/1744007
[^2]: "Cytoscape.js 2023 update: a graph theory library for visualization and analysis." Bioinformatics 39(1):btad031, 2023. https://academic.oup.com/bioinformatics/article/39/1/btad031/6988031
[^3]: Extensions listing, Cytoscape.js documentation. https://js.cytoscape.org/#extensions
[^4]: Cytoscape.js documentation (renderer/performance). https://js.cytoscape.org/#core/initialisation
[^5]: Project README, release cadence and branch model. https://github.com/cytoscape/cytoscape.js#readme

## Tags

javascript, graph-theory, network-visualization, data-visualization, canvas, bioinformatics, graph-analysis, dataviz, node-link-diagram, browser
