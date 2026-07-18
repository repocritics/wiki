# KhronosGroup/glTF

> The royalty-free "JPEG of 3D": Khronos's specification and extension registry for runtime 3D asset delivery — glTF 2.0, the format nearly every engine, viewer, and DCC tool now speaks.

[GitHub repo](https://github.com/KhronosGroup/glTF) ·
[Official website](https://www.khronos.org/gltf/) ·
License: mixed — repo docs CC-BY-4.0, spec under the Khronos Specification License (GitHub reports NOASSERTION)

## Overview

This repository is not a library — it is the home of the **glTF specification** (GL Transmission Format), its extension registry, and a large curated index of the surrounding ecosystem. glTF is a JSON-plus-binary format designed for *last-mile delivery* of 3D scenes: minimal file size, minimal runtime parsing, data laid out so a GPU can consume it with little or no transcoding[^1]. The common tagline "JPEG of 3D" captures the intent: a compact final-form distribution format, not an authoring or interchange format.

The defining tradeoff is exactly that scoping decision. glTF 2.0 deliberately fixes a single material model (PBR metallic-roughness), a single scene graph shape, and GPU-ready data layouts. That is why support is near-universal — three.js, Babylon.js, Unity, Unreal, Godot, Blender, and e-commerce viewers all consume it — but it also means glTF cannot represent arbitrary shading networks, NURBS, or non-destructive authoring state. Pipelines that need those author in USD or native DCC formats and export glTF at the end.

Development is governed by the Khronos 3D Formats Working Group. Activity here is spec activity — issues and PRs against AsciiDoc prose and JSON schemas, plus new `KHR_`/`EXT_` extensions — so the repo's 7.8k stars and 320 open issues understate real-world adoption by orders of magnitude; the format's install base is effectively "every 3D viewer shipped since ~2018". The repo remains actively maintained (pushes within days as of mid-2026), with most motion in the extension registry rather than the core 2.0 spec, which has been stable since 2017 and was ratified as ISO/IEC 12113 in 2022[^2].

## Getting Started

There is nothing to install from this repo; you consume glTF through a loader or exporter. A minimal valid glTF 2.0 asset is plain JSON:

```json
{
  "asset": { "version": "2.0" },
  "scene": 0,
  "scenes": [{ "nodes": [0] }],
  "nodes": [{ "mesh": 0 }],
  "meshes": [{ "primitives": [{ "attributes": { "POSITION": 0 } }] }]
}
```

Practical entry points:

```bash
# Inspect / optimize an asset (community CLI, widely used in pipelines)
npm install -g @gltf-transform/cli
gltf-transform inspect model.glb
```

Validate with the official Khronos glTF-Validator (drag-and-drop at github.khronos.org/glTF-Validator)[^6]. For loading: `GLTFLoader` in three.js, `SceneLoader` in Babylon.js, built-in importers in Blender, Godot, and Unreal.

## Architecture / How It Works

A glTF asset has three layers:

- **JSON scene description** — `scenes → nodes → meshes → primitives`, plus `materials`, `animations`, `skins`, `cameras`. Nodes form a tree with TRS or matrix transforms. Units are meters, Y is up.
- **Binary buffers** — raw little-endian data referenced through a `buffers → bufferViews → accessors` indirection. Accessors declare component type, count, and stride so vertex/index data can be handed to GPU APIs directly. This indirection is the format's core trick and its main source of implementation bugs (stride/offset/alignment mistakes).
- **Textures** — PNG/JPEG in core; KTX2/Basis Universal supercompressed GPU textures via `KHR_texture_basisu`[^3].

Packaging comes in three flavors: `.gltf` + external `.bin`/images, `.gltf` with base64-embedded buffers (bloats size ~33%, avoid), and `.glb`, a single binary container that is the de facto distribution form.

**Materials** are fixed-function PBR metallic-roughness in core. glTF 1.0 had instead shipped materials as literal GLSL shaders bound to WebGL — which made 1.0 unimplementable outside WebGL and is why 2.0 (June 2017) broke compatibility entirely[^4]. Post-2.0 material richness arrives through ratified `KHR_materials_*` extensions (clearcoat, transmission, volume, iridescence, anisotropy, emissive_strength), which viewers adopt at different speeds.

**Extensions** are the real evolution mechanism: `KHR_` (ratified, multi-vendor) and `EXT_`/vendor prefixes live in `extensions/` in this repo[^5]. Notable: `KHR_draco_mesh_compression` (geometry compression), `EXT_mesh_gpu_instancing`, `KHR_lights_punctual`. An asset lists `extensionsRequired` vs `extensionsUsed` — the difference decides whether an unsupporting viewer degrades gracefully or fails to load.

## Production Notes

- **glTF 1.0 is dead.** No migration path to 2.0 exists in practice; anything emitting 1.0 (very old CesiumJS pipelines) needs re-export from source.
- **Extension support is the compatibility surface.** Core 2.0 loads everywhere; `KHR_materials_transmission` or `KHR_texture_basisu` may not. Test target viewers, and prefer `extensionsUsed` + sane fallbacks over `extensionsRequired`.
- **Draco is not free.** `KHR_draco_mesh_compression` shrinks geometry 5-10x but requires shipping a WASM decoder (~300 KB) and costs decode time on load; for small meshes plain quantization (`KHR_mesh_quantization` + meshopt) often nets better end-to-end latency.
- **Texture memory is the usual killer, not geometry.** PNG/JPEG decompress to full RGBA in GPU memory; a 4096² albedo is 64 MB resident. KTX2/BasisU stays compressed on-GPU and is the single highest-leverage optimization for web/mobile delivery.
- **Validator early, validator often.** Exporters emit subtly invalid accessors that some viewers tolerate and others reject; the Khronos glTF-Validator is the arbiter[^6].
- **Don't author in glTF.** It discards construction history, modifiers, and shader graphs. Keep a source-of-truth format (Blend/USD/FBX) and treat glTF as a build artifact — round-tripping through glTF is lossy by design.
- **Animation limits.** Keyframed TRS, morph targets, and linear blend skinning only — no constraints, IK, or blending semantics; those belong to the runtime.

## When to Use / When Not

**Use when:**
- Delivering 3D content to web, mobile, AR, or e-commerce viewers.
- You want one export target that every major engine and viewer loads.
- You need compact, GPU-ready assets with a standardized PBR look.
- Building a pipeline where assets from many DCC tools must converge.

**Avoid when:**
- You need an authoring/interchange format with full fidelity — use OpenUSD or native DCC formats; glTF is a delivery endpoint.
- Your materials are custom shader graphs that PBR metallic-roughness plus ratified extensions cannot express.
- You need CAD semantics (B-rep, tolerances, PMI) — glTF stores tessellated triangles only.

## Alternatives

- PixarAnimationStudios/OpenUSD — use instead when you need composition, layering, and full-fidelity scene interchange across a DCC pipeline.
- CesiumGS/3d-tiles — use for streaming massive geospatial/city-scale content; it builds on glTF as its leaf tile format rather than replacing it.
- vrm-c/vrm-specification — use for portable VR avatars; a glTF 2.0 profile with humanoid rig and license metadata.
- assimp/assimp — use when you must import dozens of legacy formats at runtime instead of standardizing on one delivery format.
- KhronosGroup/OpenCOLLADA — the predecessor interchange format; relevant only for maintaining legacy COLLADA pipelines.

## History

| Version | Date | Notes |
|---------|------|-------|
| Drafts | 2012–2015 | Grew out of COLLADA-to-JSON / "WebGL Transmission Format" work in the Khronos COLLADA WG. |
| 1.0 | 2015-10 | First release. Materials as GLSL techniques; WebGL-bound. |
| 2.0 | 2017-06 | PBR metallic-roughness, API-neutral, GLB in core. Breaking rewrite; the enduring version[^4]. |
| — | 2018–2021 | Extension wave: Draco compression, punctual lights, KTX2/BasisU textures[^3], materials extensions. |
| AsciiDoc | 2021-09-23 | Spec source converted from Markdown to AsciiDoc[^1]. |
| ISO/IEC 12113 | 2022 | glTF 2.0 ratified as an international standard[^2]. |

## References

[^1]: KhronosGroup/glTF README — format description and AsciiDoc migration note. https://github.com/KhronosGroup/glTF/blob/main/README.md
[^2]: Khronos Group, glTF overview page (ISO/IEC 12113 status). https://www.khronos.org/gltf/
[^3]: KHR_texture_basisu extension specification. https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_texture_basisu
[^4]: glTF 2.0 specification. https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html
[^5]: glTF Extension Registry. https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md
[^6]: Khronos glTF-Validator. https://github.com/KhronosGroup/glTF-Validator

## Tags

gltf, 3d-graphics, file-format, specification, khronos, pbr, asset-delivery, webgl, standard, asset-pipeline
