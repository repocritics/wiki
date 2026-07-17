# pmndrs/drei

> A grab-bag of ready-made helpers and abstractions for react-three-fiber — the parts of a three.js scene you would otherwise re-implement on every project.

[GitHub repo](https://github.com/pmndrs/drei) ·
[Official docs](https://docs.pmnd.rs/drei) ·
[License: MIT](https://github.com/pmndrs/drei/blob/master/LICENSE)

## Overview

drei (German for "three") is a companion library to [@react-three/fiber](https://github.com/pmndrs/react-three-fiber) (R3F), the React renderer for three.js. Where R3F gives you the reconciler — the ability to describe a three.js scene as JSX — drei gives you the hundreds of small conveniences that R3F deliberately leaves out: camera controls, HTML-over-canvas overlays, GLTF loading hooks, environment lighting, instancing, text meshes, materials, gizmos, and more[^1]. It is published on npm as `@react-three/drei` and maintained by Poimandres (pmndrs), the same collective behind R3F, zustand, and jotai.

The defining characteristic of drei is that it is a *collection*, not a framework. Each export is independent, quality and maturity vary widely across the ~150 components, and there is no unifying architecture beyond "useful things for R3F." Some helpers (`OrbitControls`, `useGLTF`, `Environment`, `Html`) are battle-tested and used in effectively every R3F app; others (`Splat`, `Caustics`, `MeshTransmissionMaterial`, face-tracking helpers) wrap experimental or GPU-heavy techniques and change more often. Reading drei as "one library with one stability guarantee" is the most common mistake; it is better understood as a monorepo of loosely related snippets under a single version number.

The second defining tension is version coupling. drei sits on top of three.js, which does not treat its `examples/jsm` add-ons as a stable, semver-versioned API. To insulate itself, drei depends on [`three-stdlib`](https://github.com/pmndrs/three-stdlib) — a repackaged, typed, versioned fork of those add-ons — rather than importing from `three/examples/jsm` directly[^2]. This decision solves one class of breakage and introduces another (see Production Notes).

## Getting Started

```bash
npm install three @react-three/fiber @react-three/drei
```

drei is not standalone: `three` and `@react-three/fiber` are peer dependencies and must be installed alongside it.

```jsx
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Environment, useGLTF } from '@react-three/drei'
import { Suspense } from 'react'

function Model() {
  const { scene } = useGLTF('/model.glb')
  return <primitive object={scene} />
}

export default function App() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }}>
      <Suspense fallback={null}>
        <Model />
        <Environment preset="sunset" />   {/* IBL from a bundled HDRI */}
      </Suspense>
      <OrbitControls />                    {/* mouse orbit / zoom / pan */}
    </Canvas>
  )
}

useGLTF.preload('/model.glb')             // warm the loader cache
```

For React Native, import from `@react-three/drei/native`. That entry point omits `Html` and `Loader`, both of which depend on the DOM[^1].

## Architecture / How It Works

drei has no runtime core. It is a flat namespace of components and hooks, each authored against R3F's primitives. Three recurring implementation patterns explain most of its behavior:

- **`extend()` registration.** Several helpers teach R3F's reconciler about non-standard three.js classes by calling R3F's `extend()`, which makes new lowercase JSX elements available (for example custom materials and line geometries). This is why some drei components add global JSX intrinsics and why the corresponding TypeScript declarations occasionally lag the runtime.
- **Suspense-based loaders.** `useGLTF`, `useTexture`, `useFBX`, `useKTX2`, and friends throw promises and integrate with React `Suspense`. They cache by URL, expose `.preload()` and `.clear()`, and must be wrapped in a `<Suspense>` boundary or they will suspend the whole tree.
- **Thin wrappers over external libraries.** Many "abstractions" are ergonomic bindings to established packages: `CameraControls` wraps yomotsu/camera-controls, `Text` wraps troika-three-text, `Line` wraps meshline, GLTF Draco/KTX2 decoding pulls in the corresponding three loaders. drei's value here is the React binding and sensible defaults, not the underlying algorithm.

Because everything is layered on R3F, which is layered on three.js, drei inherits both stacks' constraints. A drei component is only ever as stable as the three.js feature it wraps, and the whole tower moves together on major version bumps.

## Production Notes

**Bundle size is the primary footgun.** drei's exports pull in heavy transitive dependencies (troika text rendering, meshline, camera-controls, postprocessing, the Draco/KTX2 decoders). Named ESM imports tree-shake well *in principle*, but importing a single component such as `Text` or `MeshTransmissionMaterial` can still drag hundreds of kilobytes into the bundle. Audit your final bundle; do not assume drei is "free" because you only imported two names.

**Version lockstep with three and R3F.** drei major versions are cut to match R3F and three major releases; you generally cannot mix a new `three` with an old drei, or vice versa. Mismatches surface as runtime errors from `three-stdlib` (a class that moved or changed signature) rather than as clean install-time peer warnings. Treat `three` + `@react-three/fiber` + `@react-three/drei` as a single upgrade unit and bump them together.

**Loaders and CDNs.** By default, Draco-compressed GLTF decoding fetches the Draco decoder from a Google-hosted CDN (gstatic), and some KTX2/basis paths behave similarly. In offline, air-gapped, or strict-CSP environments this fails silently until you self-host the decoders and point the loader at the local path.

**`Html` is expensive and DOM-only.** The `Html` component portals real DOM over the WebGL canvas and, in transform/occlusion modes, syncs position every frame. It does not exist in the `/native` build, breaks with server-side rendering unless guarded, and becomes a performance liability at high element counts.

**Experimental components move.** `MeshTransmissionMaterial`, `AccumulativeShadows`, `Caustics`, `Splat` (gaussian splatting), and the face/landmark helpers are powerful but GPU-intensive and less stable across versions. Pin your version and test after upgrades; props on these have changed between minor releases historically.

**Suspense discipline.** Every loader hook suspends. Forgetting a `<Suspense>` boundary — or placing it above expensive UI — produces either a hard throw or a fully blank scene during load. Co-locate boundaries with the components that load.

## When to Use / When Not

**Use when:**
- You are already building with react-three-fiber and want controls, loaders, lighting, and staging without hand-rolling them.
- You want proven bindings to yomotsu/camera-controls, troika text, or Draco/KTX2 loading with React ergonomics.
- You are prototyping and value breadth of ready-made pieces over minimal footprint.

**Avoid when:**
- You are not using React or R3F — drei is meaningless without them; use three.js directly.
- You ship a size-critical bundle and only need one or two behaviors; vendoring the specific upstream library (camera-controls, troika) is leaner.
- You need long-term API stability for experimental features; drei's cutting-edge components are moving targets.

## Alternatives

- pmndrs/react-three-fiber — the renderer drei extends; required underneath drei, and enough on its own for simple scenes.
- mrdoob/three.js — drop to vanilla three.js when you are not in React and want no abstraction tax.
- pmndrs/three-stdlib — use the lower-level stdlib directly when you want three's add-ons without drei's React wrappers.
- pmndrs/react-three-postprocessing — use for effect composers/post FX, which drei only lightly covers.
- brianzinn/react-babylonjs — use when you prefer the Babylon.js engine but still want a React declarative binding.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-04 | First public release alongside react-three-fiber's rise[^3]. |
| v9.x | 2023 (approx.) | Long-lived line paired with R3F v8 / three r15x era. |
| v10.x | 2024–2025 (approx.) | Aligned with R3F v9 and React 19; three-stdlib bumped in step[^2]. |

Exact per-version release dates are omitted where not verified; drei's major line is cut to track R3F and three major releases rather than on an independent cadence.

## References

[^1]: drei README and documentation — component catalog, `/native` entry point (omits `Html`/`Loader`). https://github.com/pmndrs/drei and https://docs.pmnd.rs/drei
[^2]: drei README note — the package uses the stand-alone `three-stdlib` instead of `three/examples/jsm`. https://github.com/pmndrs/three-stdlib
[^3]: Poimandres (pmndrs) organization — react-three-fiber and its ecosystem. https://github.com/pmndrs

## Tags

javascript, typescript, react, react-three-fiber, threejs, webgl, 3d, helpers, hooks, graphics, pmndrs
