# assimp/assimp

> The Open Asset Import Library: one C++ API that parses 40+ 3D file formats into a single normalized scene graph — the graphics world's default "universal importer", with per-format quality that varies accordingly.

[GitHub repo](https://github.com/assimp/assimp) ·
[Official website](https://www.assimp.org) ·
[License: modified BSD-3-Clause](https://github.com/assimp/assimp/blob/master/LICENSE)

## Overview

Assimp (Open Asset Import Library) is a C++ library, started in 2008 by Alexander Gessler and Thomas Schulze and maintained today by Kim Kulling, that imports more than 40 3D file formats — FBX, glTF 2.0, COLLADA, OBJ, STL, 3DS, IFC, 3MF, and a long tail of legacy game formats — into one unified in-memory structure, the `aiScene`[^1]. It also exports to a smaller set of formats and ships a post-processing pipeline (triangulation, normal/tangent generation, vertex dedup, mesh optimization) that turns "whatever the artist saved" into "what a GPU wants". A C API and community bindings (C#, Python, Java, Rust, JS) sit on top of the C++ core.

The defining tradeoff is breadth versus fidelity. Every format is squeezed into the same lowest-common-denominator scene model, and each importer is an independent, mostly volunteer-written parser — so quality ranges from excellent (OBJ, glTF, COLLADA) to perpetually approximate (FBX, whose reader is a from-scratch reverse-engineering of Autodesk's proprietary format, added in 3.0 without the official SDK[^2]). Assimp is what you reach for when you need *many* formats through *one* API; teams that need one format done perfectly increasingly use format-specific loaders instead — Godot notably replaced its assimp-based FBX importer with a dedicated one in 2021[^3].

At ~13.1k stars, 3.2k forks, and pushes within days of writing, the project is unambiguously alive; its ~515 open issues are the permanent backlog a 40-format parser matrix generates.

## Getting Started

```bash
# package managers
vcpkg install assimp             # Windows / cross-platform
brew install assimp              # macOS
sudo apt install libassimp-dev   # Debian/Ubuntu

# or build from source (CMake)
git clone https://github.com/assimp/assimp && cd assimp
cmake -S . -B build -DASSIMP_BUILD_TESTS=OFF && cmake --build build
```

```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>

Assimp::Importer importer;
const aiScene* scene = importer.ReadFile("model.fbx",
    aiProcess_Triangulate | aiProcess_GenSmoothNormals |
    aiProcess_JoinIdenticalVertices | aiProcess_FlipUVs);

if (!scene || (scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE) || !scene->mRootNode) {
    // importer.GetErrorString() has the parser error
}
// scene is owned by importer and freed when it goes out of scope
```

## Architecture / How It Works

Assimp is a plugin registry of parsers feeding a shared data model:

- **Importer registry.** Each format is a `BaseImporter` subclass registered in a central list. On `ReadFile`, importers are asked `CanRead` — first by extension, then by content sniffing (magic tokens) — and the first match parses the file. Format support is compile-time selectable via CMake flags (`ASSIMP_BUILD_XXX_IMPORTER=OFF`), which matters for binary size.
- **The `aiScene` model.** Everything becomes: a node hierarchy (`aiNode` with transforms), flat arrays of `aiMesh` (one material per mesh — multi-material meshes are split), key-value `aiMaterial` property sets, `aiAnimation` channels, bones with offset matrices, embedded `aiTexture`, lights, and cameras. This normalization is the product: renderers write one ingestion path. It is also lossy — format-specific concepts (FBX pivots and PreRotation, glTF PBR extensions, IFC semantics) must be flattened or approximated into the common model.
- **Post-processing pipeline.** A flag-driven chain of steps runs after parsing: `Triangulate`, `CalcTangentSpace`, `ImproveCacheLocality`, `OptimizeMeshes`/`OptimizeGraph`, `PreTransformVertices` (collapses the node hierarchy — and destroys animation, a classic footgun), `MakeLeftHanded`/`FlipUVs` for D3D conventions, `GlobalScale` for unit handling.
- **Vendored dependencies.** `contrib/` bundles zlib, pugixml, rapidjson, minizip, poly2tri, clipper, and others, compiled in by default — convenient standalone, a symbol-collision hazard when your application links its own copies. CMake options exist to prefer system libraries.

The C API (`aiImportFile` et al.) wraps the C++ core and is the basis for most language bindings, which lag the C++ API to varying degrees.

## Production Notes

- **Treat it as an offline pipeline tool, not a runtime loader.** Parsing plus post-processing is slow and allocation-heavy relative to loading a baked engine-native format. The standard production pattern is: assimp in the asset-conditioning step, custom binary blob at runtime. Shipping all importers in a runtime binary also adds megabytes; compile out what you don't use.
- **Never feed it untrusted files unsandboxed.** Forty hand-written C++ parsers are a large attack surface; assimp has a long CVE history of out-of-bounds reads/writes on malformed inputs and is continuously fuzzed under OSS-Fuzz[^4]. If users can upload models, parse in a sandboxed or privilege-separated process.
- **FBX fidelity requires an asset-corpus test suite.** Complex FBX rigs (pivot/PreRotation chains, non-uniform inherited scaling, some animation-layer setups) import approximately or wrongly in edge cases. Validate against *your* assets before committing; budget for per-asset workarounds or an offline FBX→glTF conversion step.
- **Units, axes, handedness.** Formats disagree on up-axis, handedness, and units (FBX defaults to centimeters). `aiProcess_GlobalScale` handles units; D3D consumers want `aiProcess_ConvertToLeftHanded`. A consistent 1-unit=1-meter Y-up world out of mixed sources is configuration work, not automatic.
- **Threading and ownership.** `Assimp::Importer` is not thread-safe across concurrent calls; use one instance per thread (parallel imports with separate instances are fine). The returned `aiScene` is owned by the importer — take `GetOrphanedScene()` if it must outlive the importer.
- **API stability.** Major versions break ABI and occasionally API; the 5.x line ran 2019–2024, 6.x is current. Pin a version — per-format behavior can change between minors.

## When to Use / When Not

**Use when:**
- You need to ingest many formats through one code path (editors, converters, viewers, DCC-adjacent tooling, asset pipelines).
- You want import plus GPU-prep post-processing (triangulation, tangents, cache optimization) in one dependency.
- You're on C/C++ (or accept binding lag) and can vendor a CMake project.

**Avoid when:**
- You only need glTF — cgltf or tinygltf are smaller, faster, and truer to the spec.
- FBX fidelity is the whole job — ufbx or an offline FBX→glTF conversion handles Autodesk edge cases better.
- You must parse user-uploaded models in-process without sandboxing.
- You need runtime loading performance in a shipped game — bake to an engine-native format instead.

## Alternatives

- ufbx/ufbx — single-file C FBX loader; use it when FBX correctness and robustness matter more than format breadth.
- jkuhlmann/cgltf — tiny C glTF 2.0 loader; use it when glTF is your only input format.
- syoyo/tinygltf — C++ glTF 2.0 loader/writer; same niche as cgltf with a more C++-flavored API.
- facebookincubator/FBX2glTF — offline FBX→glTF converter built on the Autodesk SDK; use it to normalize FBX out of your pipeline entirely.
- zeux/meshoptimizer — use it when you already have geometry and only need the optimization half (indexing, cache/overdraw optimization, LODs).

## History

| Version | Date | Notes |
|---------|------|-------|
| project start | 2008 | Founded by Gessler/Schulze; originally hosted on SourceForge[^1]. |
| 3.0 | 2012-07 | Own reverse-engineered FBX importer, no Autodesk SDK[^2]. |
| 3.1 | 2014-05 | Format additions, importer fixes[^5]. |
| 4.0.0 | 2017-07-18 | Modernized C++ codebase; API cleanup[^5]. |
| 4.1.0 | 2017-12-11 | glTF 2.0 support[^5]. |
| 5.0.0 | 2019-09-24 | Major release; long-running 5.x line begins[^5]. |
| 5.2.0 | 2022-01-23 | Steady bugfix cadence through 5.2.5[^5]. |
| 5.4.0 | 2024-04-07 | Last 5.x minor line[^5]. |
| 6.0.0 | 2025-05-31 | Current major line; bugfixes through 6.0.5 (2026-04)[^5]. |

## References

[^1]: Assimp official site and documentation. https://www.assimp.org — docs at https://the-asset-importer-lib-documentation.readthedocs.io/en/latest/
[^2]: Assimp supported formats list (FBX reader implemented without the Autodesk SDK). https://github.com/assimp/assimp/blob/master/doc/Fileformats.md
[^3]: Godot Engine, "FBX importer rewritten for Godot 3.2.4" — 2021-01. https://godotengine.org/article/fbx-importer-rewritten-godot-3-2-4
[^4]: OSS-Fuzz integration (`fuzz/` directory) and CVE entries for assimp. https://github.com/assimp/assimp/tree/master/fuzz — https://www.cvedetails.com/vulnerability-list/vendor_id-21610/Assimp.html
[^5]: Assimp GitHub releases (dates per release tags). https://github.com/assimp/assimp/releases

## Tags

cpp, 3d, asset-pipeline, file-formats, fbx, gltf, importer, mesh-processing, game-development, graphics
