# g-truc/glm

> A header-only C++ math library that mirrors the GLSL API — same names, same types, same conventions — for graphics code.

[GitHub repo](https://github.com/g-truc/glm) ·
[Official website](https://glm.g-truc.net) ·
[License: Happy Bunny (Modified MIT) OR MIT](https://github.com/g-truc/glm/blob/master/manual.md#section0)

## Overview

GLM (OpenGL Mathematics) is a header-only C++ library that reimplements the math primitives of the OpenGL Shading Language — `vec2`/`vec3`/`vec4`, `mat3`/`mat4`, `quat`, and the GLSL free functions (`dot`, `cross`, `normalize`, `mix`, `clamp`, etc.) — as C++ types you can use on the CPU[^1]. The design premise is that if you already know GLSL, you already know GLM: the naming and semantics match the shading-language spec deliberately, so vertex-transform code written on the CPU reads the same as the shader it feeds. First released in 2005 and hosted on GitHub since 2012[^2], it is one of the most widely vendored math libraries in the C++ graphics world, pulled in by engines, renderers, and countless OpenGL/Vulkan tutorials.

Beyond the GLSL core, GLM ships a large extension system (the `GTC`, `GTX`, and newer `EXT` families) covering matrix transforms, quaternions, projection/clip-space matrices, noise, random numbers, data packing, and easing. This is where most real graphics work happens — `glm::perspective`, `glm::lookAt`, `glm::translate/rotate/scale` all live in extensions, not the GLSL core.

The defining tension is that GLM inherits OpenGL's conventions by default — column-major matrices, column-vector multiplication (`M * v`), and a `[-1, 1]` clip-space depth range. Those defaults are wrong for Vulkan, Direct3D, and Metal, and the mismatch is the single most common source of "my scene is inverted / clipped / black" bugs. GLM makes the other conventions available, but only through preprocessor macros that must be defined consistently across the whole translation unit.

## Getting Started

GLM is header-only; there is nothing to compile. Vendor it directly, or install via a package manager:

```shell
vcpkg install glm
# or add to CMake via FetchContent / find_package(glm CONFIG REQUIRED)
```

```cpp
#include <glm/vec3.hpp>
#include <glm/mat4x4.hpp>
#include <glm/ext/matrix_transform.hpp>   // translate, rotate, scale
#include <glm/ext/matrix_clip_space.hpp>  // perspective
#include <glm/gtc/type_ptr.hpp>           // value_ptr for GL upload

glm::mat4 mvp(float translate, glm::vec2 rotate) {
    glm::mat4 proj  = glm::perspective(glm::radians(45.0f), 4.0f/3.0f, 0.1f, 100.0f);
    glm::mat4 view  = glm::translate(glm::mat4(1.0f), glm::vec3(0, 0, -translate));
    view = glm::rotate(view, rotate.y, glm::vec3(-1, 0, 0));
    view = glm::rotate(view, rotate.x, glm::vec3( 0, 1, 0));
    glm::mat4 model = glm::scale(glm::mat4(1.0f), glm::vec3(0.5f));
    return proj * view * model;   // column-vector convention: read right-to-left
}
```

Upload with `glUniformMatrix4fv(loc, 1, GL_FALSE, glm::value_ptr(m))`. Note `glm::mat4(1.0f)` constructs an identity matrix — the scalar is the diagonal, not a fill value.

## Architecture / How It Works

Everything is templates. `glm::vec3` is `glm::vec<3, float, glm::defaultp>`; `glm::mat4` is `glm::mat<4, 4, float, glm::defaultp>`. The dimension, scalar type, and precision qualifier are all template parameters, which is why `glm::dvec4` (double), `glm::ivec2` (int), and `glm::highp_vec3` all exist as thin aliases. Functions are written generically over these and specialized where SIMD applies.

**Conventions.** Matrices are stored column-major and multiply with column vectors (`result = M * v`), matching OpenGL/GLSL. Composed transforms therefore read right-to-left. Angles are radians — degrees were removed as a default years ago; use `glm::radians()` at call sites. Quaternions are constructed as `quat(w, x, y, z)` regardless of storage order, which trips up people who assume the first component is `x`.

**Extensions.** Three tiers with different stability contracts. `GTC` (core-adjacent, stable) holds the day-to-day tools: `matrix_transform`, `matrix_clip_space`, `quaternion`, `type_ptr`, `constants`, `random`. `GTX` is experimental and must be unlocked with `#define GLM_ENABLE_EXPERIMENTAL` before inclusion — its API can change between releases. The newer `EXT` headers (introduced in the 0.9.9 refactor) split functionality into fine-grained files to cut compile time[^3].

**SIMD vs constexpr.** GLM can emit SSE2–AVX2 and ARM NEON code paths, but this is *off by default*: `GLM_FORCE_INTRINSICS` enables the vectorized path, and doing so disables `constexpr` support, because the two are mutually exclusive in GLM's implementation[^4]. Aligned types (`glm::aligned_vec4`) exist to satisfy the alignment SIMD requires. Most users unknowingly run the scalar path and rely on the compiler's autovectorizer instead.

**Configuration is global.** Behavior is steered by macros — `GLM_FORCE_DEPTH_ZERO_TO_ONE`, `GLM_FORCE_LEFT_HANDED`, `GLM_FORCE_RADIANS`, `GLM_FORCE_QUAT_DATA_WXYZ`, `GLM_FORCE_CTOR_INIT`, `GLM_FORCE_SWIZZLE`. These must be defined identically in every translation unit that includes GLM; an inconsistent definition is an ODR violation that produces silent memory-layout mismatches rather than a compile error.

## Production Notes

**Vulkan/D3D depth range.** By default projection matrices map depth to `[-1, 1]`. Vulkan and Direct3D expect `[0, 1]`. Without `#define GLM_FORCE_DEPTH_ZERO_TO_ONE` your near plane is wrong and half your depth precision is wasted. Combined with Vulkan's inverted Y in clip space, this is the canonical GLM footgun — the fix is the define plus flipping `proj[1][1]`.

**Uninitialized by default.** Since 0.9.9, default-constructed vectors and matrices are *not* zero-initialized[^5] — `glm::vec3 v;` holds garbage. This was a deliberate performance change. Define `GLM_FORCE_CTOR_INIT` to restore zero-init, or always initialize explicitly. Code that upgraded from older GLM and relied on implicit zeroing broke silently.

**Compile time.** Including the umbrella `<glm/glm.hpp>` pulls in the entire GLSL surface and is slow. Prefer the granular `<glm/ext/...>` and `<glm/gtc/...>` headers for exactly what you use — the 0.9.9 header split exists specifically to let you do this. Heavy template instantiation makes GLM a measurable contributor to build times in large graphics codebases.

**SIMD is not automatic.** Assuming GLM vectorizes because it "supports SSE" is wrong; you must define `GLM_FORCE_INTRINSICS`, accept the loss of `constexpr`, and use aligned types to actually hit the fast path. For hot CPU-side transform loops, benchmark before assuming GLM is fast.

**`GTX` stability.** Experimental extensions require `GLM_ENABLE_EXPERIMENTAL` and carry no API-stability promise. `glm::decompose`, dual quaternions, and several matrix utilities live here and have historically shipped correctness fixes across point releases (e.g. `decompose` quaternion orientation in 1.0.0). Pin your version if you depend on them.

**Licensing.** GLM is dual-licensed under the "Happy Bunny License (Modified MIT)" or MIT[^6]. GitHub reports the license as `NOASSERTION` because the Happy Bunny variant is non-standard (it adds a "good, not evil" clause). If your legal review rejects non-OSI clauses, use the MIT option explicitly.

## When to Use / When Not

**Use when:**
- You're writing OpenGL/Vulkan graphics code and want CPU-side math that mirrors your shaders.
- You want a zero-dependency, header-only library that drops into any C++17 build.
- Your matrices are small and fixed-size (2×2 to 4×4) — GLM's sweet spot.
- You value a stable, GLSL-shaped API over squeezing out maximum CPU throughput.

**Avoid when:**
- You need general linear algebra — large, dynamic, or sparse matrices, decompositions, solvers (use Eigen).
- You're writing C, not C++ (use cglm).
- You're Windows/Direct3D-only and want explicit, guaranteed SIMD (DirectXMath is a closer fit).
- Peak CPU-side vector throughput is critical and you're unwilling to manage GLM's SIMD/alignment config.

## Alternatives

- libeigen/eigen — full linear-algebra library; use instead when you need dynamic-size matrices, decompositions, or solvers rather than 4×4 graphics math.
- recp/cglm — C99 port of the same GLSL-style API with mandatory SIMD; use when your codebase is C or you want vectorization without template overhead.
- microsoft/DirectXMath — SIMD-first, Windows/D3D-oriented; use when you're all-in on Direct3D and want explicit vectorized types.
- google/mathfu — game-focused SIMD math; use when you want a lean, benchmark-driven library tuned for engines.
- sgorsten/linalg — single-file, ~1k-line minimal vector/matrix header; use when GLM's size and compile cost are overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2005 | First release as OpenGL Mathematics[^2]. |
| 0.9.9.0 | 2018-05-22 | Header split into `EXT`; no default init; experimental gate `GLM_ENABLE_EXPERIMENTAL`. |
| 0.9.9.8 | 2020-04-13 | Final 0.9.9 point release; integer vector/matrix extensions. |
| 1.0.0 | 2024-01-24 | First 1.0; `constexpr` `dot`/`cross`, decompose/quaternion fixes[^7]. |
| 1.0.1 | 2024-02-26 | C++17 `[[nodiscard]]`, aligned vec3 SIMD. |
| 1.0.2 | 2025-10-15 | Packed/aligned quats, structured-bindings ext; tests off by default. |
| 1.0.3 | 2025-12-31 | Quaternion `rotate` direction revert; vec4→vec3 conversion fix[^8]. |

## References

[^1]: GLM README — "a header only C++ mathematics library for graphics software based on the OpenGL Shading Language (GLSL) specifications." https://github.com/g-truc/glm
[^2]: GLM manual, project background. https://glm.g-truc.net
[^3]: GLM 0.9.9.1 release notes — "Split headers into EXT extensions to improve compilation time #670." https://github.com/g-truc/glm/releases/tag/0.9.9.1
[^4]: GLM 0.9.9.4 release notes — "Added GLM_FORCE_INTRINSICS to enable SIMD instruction code path. By default, it's disabled allowing constexpr support." https://github.com/g-truc/glm/releases/tag/0.9.9.4
[^5]: GLM 0.9.9.0 release notes — "No more default initialization of vector, matrix and quaternion types." https://github.com/g-truc/glm/releases/tag/0.9.9.0
[^6]: GLM manual, section 0 (Licenses) — Happy Bunny License (Modified MIT) or MIT. https://github.com/g-truc/glm/blob/master/manual.md#section0
[^7]: GLM 1.0.0 release notes. https://github.com/g-truc/glm/releases/tag/1.0.0
[^8]: GLM 1.0.3 release notes. https://github.com/g-truc/glm/releases/tag/1.0.3

## Tags

cpp, header-only, graphics, mathematics, linear-algebra, opengl, vulkan, glsl, simd, quaternion, matrix, vector
