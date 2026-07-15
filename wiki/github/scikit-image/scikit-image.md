# scikit-image/scikit-image

> Image processing for the scientific Python stack — every image is a NumPy array, every algorithm a plain function.

[GitHub repo](https://github.com/scikit-image/scikit-image) ·
[Official website](https://scikit-image.org) ·
[License: BSD-3-Clause](https://github.com/scikit-image/scikit-image/blob/main/LICENSE.txt)

## Overview

scikit-image (imported as `skimage`) is a general-purpose image-processing library for Python, built directly on NumPy and SciPy. It began in 2009 as `scikits.image`, one of the SciPy "scikits" satellite projects, and was renamed to scikit-image as the ecosystem consolidated[^1]. Its reference paper was published in PeerJ in 2014[^2]. It is maintained under the Scientific Python governance umbrella and follows the SPEC recommendations for versioning and dependency support[^3].

The defining design decision is that **images are ordinary NumPy `ndarray`s and algorithms are stateless functions** organized into topical modules (`filters`, `feature`, `segmentation`, `morphology`, `transform`, `restoration`, `measure`, `registration`, `color`, `exposure`, `draw`). There is no `Image` class, no pipeline object, no lazy graph — you pass an array in and get an array (or a labelled array, or a coordinate list) out. This makes the library trivially composable with the rest of the array ecosystem (NumPy, SciPy, matplotlib, pandas, dask, napari) but means it carries none of its own I/O abstraction, GPU story, or streaming machinery.

The tension this creates: scikit-image optimizes for correctness, readability, and interoperability over raw throughput. It is the reference implementation many people learn algorithms from, but it is CPU-only and single-machine, and it has deliberately stayed on `0.x` version numbers for over fifteen years — API stability is real in practice but never contractually promised.

## Getting Started

```bash
pip install scikit-image
# or
conda install -c conda-forge scikit-image
```

```python
import numpy as np
from skimage import io, color, filters, feature

image = io.imread("photo.jpg")          # H x W x 3 uint8 ndarray
gray = color.rgb2gray(image)            # float64 in [0, 1]

edges = feature.canny(gray, sigma=2.0)  # boolean edge map

thresh = filters.threshold_otsu(gray)   # scalar
binary = gray > thresh                  # boolean mask — plain NumPy
```

Note that `rgb2gray` returns a float image in `[0, 1]`, not the `uint8` you read in. Dtype/range conversion is a recurring source of surprise (see below).

## Architecture / How It Works

scikit-image is a Python library with performance-critical inner loops written in **Cython** (compiled to C), plus some pure-NumPy vectorized implementations. There is no compiled C++ core exposed to users the way OpenCV has `cv2`; hot paths (watershed, marching cubes, local binary patterns, graph cuts, rank filters) are Cython `.pyx` modules, and everything else is Python calling NumPy/SciPy.

Key conventions that are effectively part of the architecture:

- **Coordinate order is `(row, col)` = `(y, x)`.** This matches NumPy indexing and is the opposite of OpenCV's `(x, y)`. Point coordinates, `regionprops` centroids, and transform parameters all use row-major order. Mixing scikit-image and OpenCV code without transposing is the single most common integration bug.
- **Dtype range conventions.** Functions assume a normalized range per dtype: `float` images in `[0, 1]` (or `[-1, 1]`), `uint8` in `[0, 255]`, `uint16` in `[0, 65535]`. `img_as_float`, `img_as_ubyte`, etc. convert between them *and rescale*, which silently changes pixel values if you assumed raw casting.
- **n-dimensional by default.** Most filters and morphology operations work on 2D, 3D (volumetric), and higher-dimensional arrays, not just RGB photos. This is a deliberate bias toward scientific/microscopy/medical imaging rather than consumer photo editing.
- **Lazy submodule loading.** Recent versions import submodules lazily so `import skimage` is cheap; you reach algorithms via `skimage.filters.gaussian` etc.

The build system moved from `distutils`/`setuptools` to **Meson** during the 0.20–0.22 cycle[^4], which changed how the project is compiled and packaged (relevant if you build from source or vendor it). I/O is pluggable: `skimage.io` dispatches to backends (imageio by default, plus tifffile, pillow) rather than implementing decoders itself.

## Production Notes

**It is CPU-only and single-threaded per call.** There is no GPU backend. For large images or batch throughput, the two standard escapes are: (1) `rapidsai/cucim`, which mirrors a subset of the scikit-image API on CUDA, and (2) tiling with `dask` / `dask-image` for out-of-core and multi-core execution. Do not expect real-time video rates — that is OpenCV territory.

**Dtype conversions bite in pipelines.** A function that outputs `float64` fed into one that expects `uint8` will either error or rescale unexpectedly. Standardize on one dtype early (usually `img_as_float`) and be explicit at boundaries. Many "my output is all black/white" bug reports are range mismatches.

**The `(row, col)` convention propagates.** `transform.warp`, `AffineTransform`, `measure.regionprops`, and drawing functions all use row-major coordinates. If you interoperate with libraries that use `(x, y)` (OpenCV, most plotting of raw points, GIS), you must flip axes deliberately.

**API is `0.x` and moves.** Deprecations are announced and typically honored for ~two release cycles, but parameters get renamed (`multichannel=` → `channel_axis=` was a notable ecosystem-wide change), defaults shift, and functions relocate between modules across minor versions. Pin the version in production and read the release notes before upgrading; there is no semver stability guarantee.

**Dependency support tracks SPEC 0.** The project drops old Python, NumPy, and SciPy versions on a rolling schedule[^3]. An older scikit-image will not necessarily install against the newest NumPy, and vice versa — align the whole scientific stack rather than upgrading one package.

**Build-from-source needs a toolchain.** Because of the Cython/Meson layer, installing from an sdist requires a C compiler and build dependencies. Prefer wheels (`pip`) or conda-forge; only build from source when you must.

## When to Use / When Not

**Use when:**
- You are already in the NumPy/SciPy/matplotlib world and want algorithms that compose as plain array functions.
- You need scientific imaging: segmentation, morphology, feature extraction, registration, measurement of labelled regions, n-dimensional/volumetric data.
- Readability and correctness matter more than microsecond latency (research, analysis, prototyping, teaching).
- You want a permissively licensed (BSD-3) library with no runtime service dependencies.

**Avoid when:**
- You need real-time video, webcam capture, or GPU-accelerated inference — use OpenCV or a GPU stack.
- You need deep-learning-based vision (detection, classification, segmentation networks) — this is a classical CV library, not a DL framework.
- Your workload is large enough to need out-of-core or distributed processing and you are unwilling to add dask/cucim.
- You want a consumer photo-editing API with an `Image` object and format conversions as the primary job — Pillow is a better fit.

## Alternatives

- opencv/opencv — use instead when you need real-time performance, video I/O, GPU, or the broadest catalog of classical CV algorithms; note its C++ core and `(x, y)` convention.
- python-pillow/Pillow — use instead when you only need image loading, format conversion, resizing, and basic manipulation rather than analysis.
- rapidsai/cucim — use instead (or alongside) when you have an NVIDIA GPU and want a scikit-image-compatible API at much higher throughput.
- InsightSoftwareConsortium/ITK (and SimpleITK) — use instead for heavy medical/biomedical registration and segmentation with a mature filter pipeline model.
- imageio/imageio — use instead when the task is purely reading/writing many image and video formats, with no processing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 (`scikits.image`) | 2009 | First public release as a SciPy scikit[^1]. |
| — | 2011-07 | Repository moved to GitHub; project renamed scikit-image. |
| — | 2014 | Reference paper published in PeerJ[^2]. |
| 0.19 | 2021-12 | Broad API cleanup; `channel_axis` migration underway. |
| 0.20 | 2023 | Meson build system, dropped legacy distutils path[^4]. |
| 0.22 | 2023 | Continued Meson/packaging and SPEC-0 alignment[^3]. |
| 0.24 | 2024 | Ongoing algorithm additions and deprecation cleanup. |
| 0.25 | 2024-12 | Recent stable line. |
| 0.26 | 2026 | Latest release series (per repo tags). |

Exact release dates for individual minor versions vary; consult the project's release notes for the authoritative changelog.

## References

[^1]: scikit-image project history and origins as a SciPy scikit. https://scikit-image.org/docs/stable/user_guide.html
[^2]: Stéfan van der Walt et al., "scikit-image: Image processing in Python", PeerJ 2:e453, 2014. https://doi.org/10.7717/peerj.453
[^3]: Scientific Python SPEC recommendations (SPEC 0 — minimum supported dependencies). https://scientific-python.org/specs/
[^4]: scikit-image install/build documentation (Meson build backend). https://github.com/scikit-image/scikit-image/blob/main/INSTALL.rst

## Tags

python, image-processing, computer-vision, numpy, scipy, scientific-computing, cython, segmentation, morphology, bsd-licensed
