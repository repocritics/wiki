# opencv/opencv

> The de facto classical computer-vision library — 2500+ image and video algorithms with a C++ core and bindings for Python, Java, and JavaScript.

[GitHub repo](https://github.com/opencv/opencv) ·
[Official website](https://opencv.org) ·
[License: Apache-2.0](https://github.com/opencv/opencv/blob/4.x/LICENSE)

## Overview

OpenCV (Open Source Computer Vision Library) is the oldest and most widely deployed general-purpose computer-vision library. It began at Intel around 1999–2000 under Gary Bradski, reached a 1.0 release in 2006, and has been community-governed under the OpenCV Foundation (now OpenCV.org) since 2012[^1]. The core is C++; the day-to-day audience is Python developers who use the `cv2` bindings on top of NumPy arrays.

Its scope is enormous and mostly *classical*: image I/O and codecs, color conversion, filtering, morphology, feature detection (ORB, SIFT, AKAZE), camera calibration and stereo, contour and shape analysis, optical flow, background subtraction, image stitching, and photogrammetry. A `dnn` module runs (does not train) neural networks imported from ONNX, TensorFlow, Caffe, and Darknet. The defining tension of the project is age: OpenCV predates the deep-learning era, and its API carries two decades of accreted conventions — C-style enums, the infamous BGR channel order, in-place output arguments, and a `dnn` module that is a bolt-on rather than a native design.

Licensing changed direction with the 5.x line: OpenCV was BSD-3-Clause for its entire 1.x–4.x history and is being relicensed to Apache-2.0 going forward[^2]. GitHub currently reports the repository as Apache-2.0; verify the `LICENSE` file for the exact version you vendor, because 4.x point releases in the field are still commonly BSD-3-Clause.

## Getting Started

```bash
# Python — community-maintained wheels, the standard install path
pip install opencv-python            # core + GUI
# pip install opencv-contrib-python  # + extra/patented modules
# pip install opencv-python-headless # servers/CI, no GUI (no cv2.imshow)
```

```python
import cv2

img = cv2.imread("input.jpg")          # BGR order, returns None on failure (no exception)
if img is None:
    raise FileNotFoundError("input.jpg")

gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
edges = cv2.Canny(gray, 100, 200)
cv2.imwrite("edges.png", edges)
```

The `opencv-python` wheels are built and published by a community maintainer, not the OpenCV core team — the project itself ships only source[^3]. For CUDA, non-free codecs, or custom modules you build from source with CMake against `opencv_contrib`.

## Architecture / How It Works

OpenCV is a set of independently compiled **modules** that share one N-dimensional dense array type, `cv::Mat` (exposed to Python as a NumPy array). Core modules include `core` (Mat, arithmetic), `imgproc` (filtering, geometry, color), `imgcodecs`/`videoio` (file and camera I/O), `calib3d`, `features2d`, `objdetect`, `photo`, `stitching`, `ml`, and `dnn`. Extra and less-stable modules live in a separate repository, `opencv_contrib`, and must be compiled in explicitly.

- **The `Mat` and reference semantics.** `cv::Mat` uses reference-counted shared data. Copying a `Mat` copies a header, not pixels; `clone()` deep-copies. Many functions take a pre-allocated output array (`OutputArray`) and reuse its buffer. This is fast but a frequent source of aliasing surprises.
- **T-API / transparent OpenCL.** Since 3.0, functions accept `UMat` in place of `Mat` and transparently dispatch to OpenCL on a GPU when one is available, falling back to CPU otherwise[^4]. This is separate from the CUDA modules (`cudaimgproc` etc.) in `opencv_contrib`, which are an entirely different, explicit GPU path.
- **The `dnn` module** is an inference engine only. It imports frozen graphs and runs forward passes with pluggable backends: the built-in CPU implementation, CUDA, or Intel OpenVINO. It does not train and lags framework-native runtimes on newer operators.
- **Bindings are generated.** The Python, Java, and JS (`opencv.js`, via Emscripten/WASM) APIs are produced by parsers over the C++ headers, so binding coverage tracks the C++ surface but idioms (BGR, output args) leak straight through.

## Production Notes

**BGR, not RGB.** `imread` and `VideoCapture` return channels in blue-green-red order for historical reasons. Every hand-off to Matplotlib, PIL, PyTorch, or any model trained on RGB needs an explicit `cvtColor(..., COLOR_BGR2RGB)`. This is the single most common OpenCV bug.

**Silent failures.** `imread` returns `None` (not an exception) when a path is wrong or a codec is missing; `VideoCapture.read()` returns `(False, None)` at end-of-stream or on a dropped frame. Always check return values.

**The three-package trap.** `opencv-python`, `opencv-contrib-python`, and `-headless` all install the same `cv2` module. Installing more than one, or mixing them across transitive dependencies, produces an undefined winner. Pin exactly one, and prefer `-headless` in Docker/CI where no display exists.

**pip wheels are limited by design.** The published wheels have no CUDA, ship a bundled FFmpeg with no proprietary codecs, and disable some GUI backends. Hardware-accelerated decode, GStreamer pipelines, or the CUDA modules require a from-source CMake build — a genuinely involved undertaking on Windows and for cross-compilation.

**GUI and threading.** `imshow`/`waitKey` must run on the main thread (hard requirement on macOS) and are absent in headless builds. `waitKey` is also the event pump — windows do not repaint without it.

**Video I/O is the flaky part.** Backend behavior (`CAP_FFMPEG`, `CAP_GSTREAMER`, `CAP_V4L2`, `CAP_DSHOW`) varies by platform and build; frame counts, seeking, and timestamps from network/variable-frame-rate streams are unreliable. Many production pipelines call FFmpeg directly and hand frames to OpenCV.

**Memory and perf.** Large images are dense CPU arrays with no automatic GPU offload; batch pipelines are CPU- and RAM-bound unless you opt into `UMat`/CUDA. Most `imgproc` functions are multithreaded internally (TBB/OpenMP/pthreads depending on build).

## When to Use / When Not

**Use when:**
- You need classical CV — calibration, geometry, features, contours, optical flow, stitching — where there is no reason to reach for a neural network.
- You need broad, battle-tested image/video I/O and preprocessing feeding another system.
- You want one library that runs the same algorithms across C++, Python, Java, and the browser.
- You need to run an existing exported model on CPU or via OpenVINO without pulling in a full DL framework.

**Avoid when:**
- Your workload is training or heavy deep-learning inference — use PyTorch/torchvision or ONNX Runtime; `dnn` is inference-only and trails on new operators.
- You only need basic load/resize/crop/save — Pillow or scikit-image is lighter and more Pythonic.
- You want differentiable, GPU-native image ops inside a training loop — Kornia is purpose-built for that.
- You need reliable, seekable video decoding of arbitrary formats — drive FFmpeg directly.

## Alternatives

- scikit-image/scikit-image — use instead when you want pure-Python, NumPy-native image processing with a cleaner API and no C++ build concerns.
- python-pillow/Pillow — use instead for straightforward image loading, format conversion, resizing, and drawing without CV algorithms.
- kornia/kornia — use instead when you need differentiable, GPU-accelerated CV operators inside a PyTorch training pipeline.
- pytorch/vision (torchvision) — use instead for modern deep-learning vision models, datasets, and transforms.
- opencv/opencv_contrib — not an alternative but the required companion repo for extra/non-free modules (CUDA, `xfeatures2d`, `aruco` in older lines).

## History

| Version | Date | Notes |
|---------|------|-------|
| alpha | 2000 | First public release from Intel Research[^1]. |
| 1.0 | 2006-10 | First stable release; C API. |
| 2.0 | 2009-09 | Modern C++ API and `cv::Mat` introduced. |
| 2.4 | 2012 | Long-lived stable line; project moves to community governance[^1]. |
| 3.0 | 2015-06 | `opencv_contrib` split out; transparent OpenCL (T-API)[^4]. |
| 3.3 | 2017-08 | `dnn` module promoted from contrib into the main repo. |
| 4.0 | 2018-11 | C++11 baseline; API cleanup; DNN and QR improvements[^5]. |
| 4.4 | 2020-07 | SIFT moved to the main module after its patent expired. |
| 4.x | ongoing | Current default branch; 5.x in development with the Apache-2.0 relicense[^2]. |

## References

[^1]: OpenCV, "About / History." https://opencv.org/about/
[^2]: OpenCV, "OpenCV To Move To Apache 2 License" (announced for OpenCV 5). https://opencv.org/blog/
[^3]: opencv-python project (community-maintained wheels), README on packaging and build options. https://github.com/opencv/opencv-python
[^4]: OpenCV docs, "Transparent API (T-API) and OpenCL." https://docs.opencv.org/4.x/
[^5]: OpenCV, "OpenCV 4.0" release notes. https://opencv.org/opencv-4-0/

## Tags

c-plus-plus, python, computer-vision, image-processing, video-processing, deep-learning-inference, opencv, machine-vision, cross-platform, apache-2
