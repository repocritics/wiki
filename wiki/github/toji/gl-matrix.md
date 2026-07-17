# toji/gl-matrix

> Hand-tuned JavaScript vector and matrix math for WebGL, built around an out-parameter API that trades ergonomics for near-zero allocation.

[GitHub repo](https://github.com/toji/gl-matrix) ·
[Official website](https://glmatrix.net) ·
[License: MIT](https://github.com/toji/gl-matrix/blob/master/LICENSE.md)

## Overview

glMatrix is a small, dependency-free math library for 2D/3D graphics in the
browser, written by Brandon Jones (`toji`) and first published in 2011[^1]. It
supplies the linear-algebra primitives that JavaScript lacks — `vec2`, `vec3`,
`vec4`, `mat2`, `mat2d`, `mat3`, `mat4`, `quat`, and `quat2` (dual quaternions)
— in a form shaped entirely by one goal: run fast enough to be called every
frame without generating garbage.

The library's defining decision is its calling convention. Nearly every
operation takes an explicit output object as its first argument —
`vec3.add(out, a, b)` writes into `out` and returns it — so the caller owns and
reuses the memory. There are no classes and no operator overloading; a "vector"
is just a `Float32Array` of length 3. This is deliberately un-ergonomic in
exchange for predictable performance: a well-written render loop allocates
nothing, which keeps the garbage collector from introducing frame stalls.

That tradeoff is also the source of most confusion for new users. Code reads as
long sequences of verbose calls, temporaries must be pre-allocated by hand, and
the `Float32Array` default silently reduces precision. glMatrix is not trying to
be pleasant; it is trying to be the thing a game engine or WebGL demo calls
underneath a nicer API. For that job it has been a de facto standard for over a
decade.

## Getting Started

```bash
npm install gl-matrix
```

```js
import { mat4, vec3 } from "gl-matrix";

// Build a model-view-projection matrix for a WebGL draw call.
const proj = mat4.create();               // identity 4x4 (column-major)
mat4.perspective(proj, Math.PI / 4, 16 / 9, 0.1, 100);

const view = mat4.create();
mat4.lookAt(view, [0, 0, 5], [0, 0, 0], [0, 1, 0]);

const mvp = mat4.create();
mat4.multiply(mvp, proj, view);           // mvp = proj * view

// Reuse buffers in a hot loop — no per-frame allocation.
const axis = vec3.fromValues(0, 1, 0);
function rotate(model, radians) {
  mat4.rotate(model, model, radians, axis); // out === input is allowed here
}
```

The whole-namespace `import { mat4 }` form is tree-shakeable as of v3, so
unused modules are dropped by bundlers.

## Architecture / How It Works

glMatrix is a flat collection of stateless module functions, not an object
model. Each module (`vec3`, `mat4`, …) is a plain namespace of functions that
operate on typed arrays. There is no `this`, no prototype chain, and no hidden
state beyond one global setting (described below). This flatness is what makes
the functions cheap to inline and monomorphic for the JIT.

Key internals worth knowing:

- **Column-major matrices.** `mat4` stores its 16 elements in column-major
  order to match OpenGL/WebGL's `uniformMatrix4fv` expectations, so a matrix can
  be handed to the GPU without transposition.
- **The `out` parameter and aliasing.** Most functions permit the output to
  alias an input (`vec3.add(a, a, b)`), and the hand-written bodies read all
  inputs into locals before writing results to make this safe. Not every
  function documents aliasing guarantees, so mixing `out` with inputs on less
  common operations warrants a check against the source.
- **`glMatrix.setMatrixArrayType`.** A single global switch chooses the backing
  array type for everything created afterward — `Float32Array` by default, or
  `Array` for double precision. It affects only objects created *after* the
  call, which makes toggling it mid-program a subtle footgun.
- **`glMatrix.EPSILON`.** Equality helpers (`vec3.equals`, etc.) compare within
  a tolerance rather than bit-exact, because float accumulation never lands on
  exact values.

The v2-to-v3 transition was the library's one major rearchitecture: v3 moved to
ES modules with named exports, became tree-shakeable, and changed or removed
several v2 behaviors. Code written against v2's single global `glMatrix` object
does not run unmodified on v3.

## Production Notes

**`Float32Array` precision is the number-one surprise.** The default 32-bit
backing store carries roughly 7 significant decimal digits. Large world
coordinates, long transform chains, and accumulated rotations visibly jitter or
drift. The documented remedy is `glMatrix.setMatrixArrayType(Array)`, which
switches to 64-bit doubles and, on modern engines, is frequently *faster* as
well as more accurate — the README itself recommends it for performance[^2].
Set it once at startup, before creating any matrices.

**Allocation discipline is on you.** The out-parameter API only pays off if you
reuse buffers. The idiomatic-looking `const c = vec3.add(vec3.create(), a, b)`
allocates a new array every call and reintroduces exactly the GC pressure the
design exists to avoid. Hot paths should hoist scratch vectors/matrices out of
the loop.

**No classes means no method chaining and no type safety at call sites.**
Everything is a bare `Float32Array`, so a `vec3` and a `vec4` are
indistinguishable to the type system and passing the wrong length is a silent
logic bug, not an error. TypeScript users get the shipped `.d.ts` types, but
they cannot catch length mismatches.

**Quaternion drift.** Repeatedly multiplying quaternions accumulates
non-unit-length error; call `quat.normalize` periodically when composing many
rotations, as glMatrix does not renormalize for you.

**Maintenance cadence is slow and that is mostly fine.** The v3 line is stable
and the API surface has been static for a long time; this is a mature,
low-churn dependency rather than an abandoned one. A class-based, TypeScript
rewrite for a future major version has been discussed and worked on publicly,
but treat any specific v4 timeline as unverified before you depend on it.

## When to Use / When Not

**Use when:**
- You are writing WebGL/WebGPU code directly and need matrices/quaternions that
  hand off to the GPU without conversion.
- You want zero dependencies and full control over allocation in a render loop.
- You need a small, well-understood, battle-tested math core under your own
  abstraction.

**Avoid when:**
- You want an object-oriented, chainable API (`v.add(b).normalize()`) — the
  out-parameter style will feel hostile.
- You are already inside a framework that ships its own math (three.js, Babylon)
  — use theirs to avoid two coordinate conventions in one codebase.
- Double precision everywhere matters more than raw throughput and you would
  rather not manage the global array-type switch.

## Alternatives

- mrdoob/three.js — bundles class-based `Vector3`/`Matrix4`/`Quaternion`; use it
  when you are already rendering with three.js and want one math convention.
- uber-web/math.gl — class-based, `Float64Array`-friendly math for the vis.gl /
  deck.gl stack; use when you want OO ergonomics and 64-bit precision by default.
- greggman/wgpu-matrix — a lean, WebGPU-oriented library in the same functional
  spirit; use for new WebGPU projects wanting a modern take on the same idea.
- stackgl/gl-vec3 (and gl-mat4, gl-vec4) — glMatrix's functions republished as
  one npm package per module; use when you want maximally granular imports.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011 | First release by Brandon Jones for WebGL demos[^1]. |
| 2.x | ~2014–2018 | Single global `glMatrix` object; not tree-shakeable. |
| 3.0.0 | 2019 | ES-module rewrite, tree-shakeable, breaking API changes[^3]. |
| 3.4.x | ~2022 | Current stable line; low-churn maintenance. |

## References

[^1]: glMatrix homepage and history. https://glmatrix.net/
[^2]: gl-matrix README — recommends `glMatrix.setMatrixArrayType(Array)` for performance in modern browsers. https://github.com/toji/gl-matrix/blob/master/README.md
[^3]: gl-matrix releases (v3 ES-module rewrite). https://github.com/toji/gl-matrix/releases

## Tags

javascript, webgl, linear-algebra, matrix, vector, quaternion, 3d-graphics, math, gamedev, typed-arrays
