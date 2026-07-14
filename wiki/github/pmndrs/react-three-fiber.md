# pmndrs/react-three-fiber

> A React renderer for three.js — express a WebGL scene as declarative, stateful JSX components.

[GitHub repo](https://github.com/pmndrs/react-three-fiber) ·
[Official website](https://docs.pmnd.rs/react-three-fiber) ·
[License: MIT](https://github.com/pmndrs/react-three-fiber/blob/master/LICENSE)

## Overview

react-three-fiber (published as `@react-three/fiber`, commonly "R3F") is a custom React
reconciler that renders to a three.js scene graph instead of the DOM[^1]. It is maintained
by Poimandres (pmndrs), the collective behind zustand, jotai, valtio, react-spring, and drei.
Rather than wrap three.js in a component library, R3F implements React's reconciler protocol:
`<mesh />` in JSX literally constructs `new THREE.Mesh()`, and any three.js class is usable as
a JSX element without a binding being written for it. New three.js releases are therefore
available immediately — the renderer does not enumerate three's API, it maps onto it generically.

The defining tension is that R3F is a *thin* layer over a *large* imperative library. You get
React's component model, hooks, and ecosystem (suspense, context, state libraries) for scene
construction, but you still must understand three.js: cameras, geometries, materials, lights,
disposal, and the WebGL render loop. R3F does not abstract three.js away — it exposes it through
a different authoring surface. Teams that expect a high-level 3D toolkit are often surprised by
how much three.js knowledge remains mandatory.

The per-frame render loop runs *outside* React. `useFrame` callbacks mutate three.js objects
directly on each animation frame and never trigger React re-renders; React reconciliation is
reserved for structural scene changes. This is what keeps R3F competitive with hand-written
three.js at scale, but it means two mental models coexist in one codebase: declarative React
state for structure, imperative mutation for animation.

## Getting Started

```bash
npm install three @react-three/fiber
# TypeScript users also want @types/three
```

R3F is a React renderer and must be paired with a matching React major: `@react-three/fiber@8`
pairs with React 18, `@react-three/fiber@9` pairs with React 19[^2].

```jsx
import { createRoot } from 'react-dom/client'
import { useRef, useState } from 'react'
import { Canvas, useFrame } from '@react-three/fiber'

function Box(props) {
  const ref = useRef()                          // direct handle to the THREE.Mesh
  const [hovered, hover] = useState(false)
  useFrame((state, delta) => (ref.current.rotation.x += delta)) // runs outside React
  return (
    <mesh {...props} ref={ref} scale={hovered ? 1.5 : 1}
      onPointerOver={() => hover(true)}
      onPointerOut={() => hover(false)}>
      <boxGeometry args={[1, 1, 1]} />          {/* args = constructor arguments */}
      <meshStandardMaterial color={hovered ? 'hotpink' : 'orange'} />
    </mesh>
  )
}

createRoot(document.getElementById('root')).render(
  <Canvas>
    <ambientLight intensity={Math.PI / 2} />
    <Box position={[-1.2, 0, 0]} />
    <Box position={[1.2, 0, 0]} />
  </Canvas>,
)
```

## Architecture / How It Works

R3F is built on the `react-reconciler` package — the same host-config mechanism `react-dom` and
`react-native` use[^1]. Its host config defines how to create instances (`new THREE.*`), attach
them to parents (`scene.add(child)`), apply props, and remove them. Key mechanics:

- **The catalog.** Every three.js export is registered as a lowercase JSX element (`THREE.Mesh`
  → `<mesh>`, `THREE.MeshStandardMaterial` → `<meshStandardMaterial>`). Custom or third-party
  classes are added with `extend({ MyClass })`, after which `<myClass>` becomes valid. In v9 this
  is also exposed via the `extend` API with improved TypeScript inference[^3].
- **`args`** is the constructor argument array, applied once at instantiation. Changing `args`
  reconstructs the object (three.js constructors are not re-runnable).
- **`attach`.** Props like `geometry` and `material` on a mesh are set by *attaching* a child to a
  named parent property rather than calling `.add()`. `<boxGeometry />` inside `<mesh>` sets
  `mesh.geometry`; the reconciler infers common attach points and lets you override with `attach`.
- **Set-on-property convention.** Any prop maps to a property path on the object:
  `position={[x,y,z]}` calls `object.position.set(x,y,z)`; `material-color="red"` sets
  `object.material.color`. This "pierced" prop syntax is how R3F stays generic.
- **The render loop.** `<Canvas>` sets up the renderer, a default camera, a resize observer, and
  a `requestAnimationFrame` loop. By default it renders on-demand only when React commits or a
  `useFrame` subscriber exists; `frameloop="demand"` renders only on invalidation, which matters
  for battery and idle CPU.

Because the loop and object mutation live outside React's render phase, animation does not incur
reconciliation cost. React is used for what it is good at — diffing a tree of structure — and
raw three.js mutation handles the 60fps path.

## Production Notes

**`useFrame` bypasses React state.** Mutating `ref.current` in `useFrame` is the intended pattern
and is fast, but it means those changes are invisible to React. Mixing frequent `setState` into
the render loop reintroduces reconciliation on every frame and destroys performance. The rule in
practice: structure via state, motion via refs/mutation.

**Disposal and memory leaks.** three.js geometries, materials, and textures hold GPU resources
that are not garbage-collected. R3F auto-disposes objects it created when they unmount, but shared
or externally-created resources (e.g. from `useLoader`, or a material reused across meshes) are
not, and manual `.dispose()` or `dispose={null}` handling is required. Leaked GPU memory from
re-mounting scenes is one of the most common R3F production bugs.

**Everything is three.js underneath.** Debugging often means dropping into three.js concepts —
frustum culling, draw calls, texture formats, `sRGB` vs linear color spaces (three.js changed its
default color management, which R3F follows), and shadow map configuration. R3F does not shield you
from any of this; profiling tools are three.js/WebGL tools (spector.js, the browser's WebGL
inspector), not React DevTools.

**The ecosystem is where the leverage is.** Bare R3F is intentionally minimal. Nearly every real
project also pulls in `@react-three/drei` (cameras, controls, loaders, helpers — effectively a
second, much larger dependency), plus domain packages: `@react-three/postprocessing`, `rapier`
or `cannon` for physics, `@react-three/xr` for VR/AR, `gltfjsx` to convert models to JSX. Budget
for the fact that "using R3F" usually means using several pmndrs packages whose versions must stay
compatible with each other, with R3F, and with the pinned three.js version.

**Upgrade pain is version-triple coupled.** An R3F upgrade is really a coordinated bump of React,
three.js, and every `@react-three/*` package at once. v9's move to React 19 forced the ecosystem
(drei, postprocessing, etc.) to release matching majors, and third-party libraries lagged on peer
ranges. v9 also revised the TypeScript surface: element typings moved to `ThreeElements` and the
old `JSX.IntrinsicElements` augmentation approach changed, so TS-heavy codebases saw type churn[^3].

**SSR and bundling.** `<Canvas>` requires a browser (WebGL context); it must be client-only under
Next.js App Router (`"use client"` and typically dynamic import with `ssr: false`). three.js is a
large dependency — code-splitting the 3D route is standard practice.

## When to Use / When Not

**Use when:**
- You already use React and want 3D that participates in your component tree, state, and suspense.
- You want three.js's full capability but authored declaratively and composably.
- You are building configurators, product viewers, data visualization, or interactive scenes where
  UI state and 3D state are tightly linked.
- You want the pmndrs ecosystem (drei, physics, postprocessing) as ready-made building blocks.

**Avoid when:**
- You don't use React — the entire value proposition is the React model; use three.js directly, or
  a renderer for your framework.
- You need a high-level engine with a scene editor, asset pipeline, and gameplay systems — R3F is a
  renderer, not a game engine (consider a full engine instead).
- Your team doesn't want to learn three.js — R3F reduces boilerplate but not the underlying concepts.
- You need heavy SSR/static rendering of the 3D content itself; WebGL is inherently client-side.

## Alternatives

- threejs/three.js — the library R3F renders to; use directly when you don't want React or need
  full imperative control with no reconciler overhead.
- pmndrs/drei — not an alternative but the near-mandatory companion helper library for R3F.
- threlte/threlte — three.js renderer for Svelte; use when your stack is Svelte, not React.
- brianzinn/react-babylonjs — same declarative idea over Babylon.js; use when you want Babylon's
  built-in engine features (physics, GUI, asset system) over three.js.
- aframevr/aframe — HTML/entity-component framework over three.js; use for WebXR/declarative markup
  without a JS component model.
- troisjs/trois — Vue-flavored three.js components; use when your framework is Vue.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2019 | First release as `react-three-fiber` reconciler for three.js[^1]. |
| v5 | 2020 | Rename to `@react-three/fiber` under the pmndrs scope; ecosystem consolidation. |
| v6 | 2021 | Reworked `<Canvas>`, on-demand rendering, events/raycasting overhaul. |
| v7 | 2021 | Reconciler and typing improvements. |
| v8 | 2022 | React 18 support; React Native target (`@react-three/fiber/native`)[^2]. |
| v9 | 2025 | React 19 support; revised TypeScript element typings (`ThreeElements`)[^3]. |

## References

[^1]: react-three-fiber documentation — "Introduction" and the reconciler model. https://docs.pmnd.rs/react-three-fiber
[^2]: `@react-three/fiber` README — React major pairing (v8↔React 18, v9↔React 19) and React Native usage. https://github.com/pmndrs/react-three-fiber
[^3]: react-three-fiber v9 migration guide — React 19 and TypeScript `ThreeElements` changes. https://docs.pmnd.rs/react-three-fiber/tutorials/v9-migration-guide

## Tags

typescript, react, threejs, webgl, 3d, renderer, react-reconciler, declarative, graphics, pmndrs
