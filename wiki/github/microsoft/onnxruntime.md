# microsoft/onnxruntime

> Cross-platform runtime that executes ONNX models, with a pluggable execution-provider layer that maps one graph onto many accelerators.

[GitHub repo](https://github.com/microsoft/onnxruntime) ·
[Official website](https://onnxruntime.ai) ·
[License: MIT](https://github.com/microsoft/onnxruntime/blob/main/LICENSE)

## Overview

ONNX Runtime (ORT) is Microsoft's inference and training engine for models in the ONNX (Open Neural Network Exchange) format[^1]. A model trained in PyTorch, TensorFlow/Keras, scikit-learn, LightGBM, or XGBoost is exported to a single `.onnx` graph, and ORT runs that graph on whatever hardware is available. It is written in C++ with first-party bindings for Python, C, C++, C#, Java, JavaScript/Node, and Objective-C. First released in late 2018, it is now the default inference path behind Windows ML, large parts of Office and Bing, and PyTorch's own ONNX export tooling.

The defining idea is the **Execution Provider (EP)** abstraction. The same ONNX graph can be dispatched to CPU (the built-in MLAS kernels), CUDA, TensorRT, DirectML, OpenVINO, CoreML, QNN, ROCm, or the browser (WASM + WebGPU/WebNN) without changing the model. This is ORT's main reason to exist: it decouples "the model" from "the silicon," which is exactly the portability that a framework-native runtime (TorchScript, TFLite) does not give you across vendors.

The tension is that portability is not uniform. Each EP supports a different subset of ONNX operators, so a graph is partitioned across whatever the chosen EP can handle, with everything else falling back to CPU. The abstraction is real, but the performance you get is highly dependent on how cleanly your specific model maps onto your specific EP — and that mapping is where most operator effort goes.

## Getting Started

```bash
# CPU build (do NOT also install onnxruntime-gpu — they conflict)
pip install onnxruntime
# or, for NVIDIA GPUs:
pip install onnxruntime-gpu
```

```python
import onnxruntime as ort
import numpy as np

# Provider list is ordered by priority; ORT falls back left-to-right.
sess = ort.InferenceSession(
    "model.onnx",
    providers=["CUDAExecutionProvider", "CPUExecutionProvider"],
)

# Feed by the graph's declared input names.
inputs = {sess.get_inputs()[0].name: np.random.randn(1, 3, 224, 224).astype(np.float32)}
outputs = sess.run(None, inputs)
print(outputs[0].shape)
```

Which providers actually loaded is worth checking explicitly — a silently-missing CUDA build will run on CPU without erroring:

```python
print(sess.get_providers())  # verify CUDA is actually active
```

## Architecture / How It Works

ORT loads an ONNX model (a protobuf graph of operators at a declared **opset** version) and runs it through three stages:

1. **Graph optimization** — level-based transforms (`ORT_ENABLE_BASIC` / `EXTENDED` / `ALL`) do constant folding, redundant-node elimination, and operator fusion (e.g. Conv+BatchNorm, attention fusion for transformers). Optimized graphs can be serialized so the cost is paid once.
2. **Partitioning** — the optimizer walks the graph and assigns each node to the highest-priority EP that claims support for it. Contiguous runs of nodes on one EP become a subgraph; boundaries between EPs become device-copy points.
3. **Execution** — each subgraph runs on its EP's kernels. The CPU EP uses MLAS, ORT's own SIMD kernel library; accelerator EPs either implement kernels directly (CUDA) or hand the whole subgraph to a vendor compiler (TensorRT, OpenVINO, QNN) that builds an optimized engine.

The EP boundary is the load-bearing internal contract. A "compiling" EP like TensorRT ingests an entire subgraph, builds a hardware-specific engine (which can take minutes on first run), and caches it. A "kernel" EP like CUDA dispatches op-by-op. This difference explains most of ORT's performance surprises: the same model can be fast on one EP and slow on another purely because of how much of the graph the EP could absorb versus how much bounced back to CPU.

Beyond core inference, ORT ships several derived surfaces from the same tree: **ORT Web** (WASM/WebGPU for the browser), **ORT Mobile** and the reduced `.ort` format (a trimmed binary with only the operators your model needs), quantization tooling (dynamic and static INT8, the QDQ format), and **ORT training** (`ORTModule`, a drop-in wrapper that accelerates PyTorch training, plus on-device training). LLM-specific serving lives in the separate `onnxruntime-genai` repo, not here.

## Production Notes

**The `onnxruntime` vs `onnxruntime-gpu` pip trap.** These two packages install the same module name and must not coexist; having both in an environment produces nondeterministic load behavior. Pick one per environment.

**CUDA/cuDNN version pinning.** Each `onnxruntime-gpu` release is built against specific CUDA and cuDNN versions. A mismatch does not fail loudly at install — it fails at session creation with an opaque provider-load error, or silently falls back to CPU. Always cross-check the release notes' CUDA/cuDNN matrix against your driver stack, and verify `get_providers()` at runtime.

**Silent CPU fallback is the number-one perf footgun.** If your requested EP fails to load, or a chunk of your graph has ops the EP does not implement, that work runs on CPU. On a GPU EP this also inserts host↔device memory copies at every partition boundary, which can make an "accelerated" model slower than pure CPU. Inspect partitioning with the ORT profiler (`enable_profiling`) or session logging before trusting a speedup.

**IOBinding for GPU.** By default ORT copies inputs from host to device and outputs back every call. For throughput-sensitive GPU serving, use `IOBinding` to keep tensors resident on the device and eliminate per-call copies.

**Threading.** `intra_op_num_threads` and `inter_op_num_threads` default to heuristics that often oversubscribe in containerized or multi-tenant deployments. Under a process-per-request server, set them low (often 1) to avoid thread thrashing across concurrent sessions.

**TensorRT EP warmup.** Engine builds happen on first inference and can take minutes; enable the engine cache (`trt_engine_cache_enable`) and warm the model at startup, or first-request latency will be catastrophic. Cached engines are tied to the exact GPU, driver, and ORT/TRT versions.

**Opset drift on export.** The most common correctness bug is exporting from PyTorch at an opset ORT does not fully support, or with dynamic axes the target EP cannot handle. Validate exported graphs with `onnx.checker` and run a numerical parity check against the source framework before shipping.

## When to Use / When Not

**Use when:**
- You need one model artifact to run across CPU, multiple GPU vendors, mobile, and the browser.
- You want to decouple training framework choice from the deployment runtime.
- You're shipping classical ML (scikit-learn / XGBoost / LightGBM) to production and want a single fast serving path via the ONNX converters.
- You need in-browser or on-device inference with a small, operator-trimmed binary.

**Avoid when:**
- You're staying entirely inside PyTorch and `torch.compile` already meets your latency goals — the export + parity-check overhead buys you nothing.
- You target a single NVIDIA deployment and want maximum throughput — raw TensorRT or a purpose-built server may beat ORT's TensorRT EP with less abstraction.
- Your model uses cutting-edge or custom operators that lack ONNX equivalents; export will fail or require hand-written custom ops.
- Your workload is LLM text generation — reach for a serving stack built for KV-cache and batching rather than generic ORT.

## Alternatives

- pytorch/pytorch — use its native `torch.compile` / TorchScript when you never leave the PyTorch ecosystem and don't need cross-vendor portability.
- NVIDIA/TensorRT — use directly when you deploy only on NVIDIA GPUs and want peak throughput over portability.
- openvinotoolkit/openvino — use when your fleet is Intel CPU/iGPU/NPU and you want Intel's own optimizer rather than ORT's OpenVINO EP.
- tensorflow/tensorflow (TFLite) — use for mobile/edge when your model already lives in the TensorFlow ecosystem.
- ggerganov/llama.cpp — use for on-device LLM inference, where GGUF + specialized quantization beats a generic ONNX path.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.4 | 2018-12 | First public release; CPU + CUDA inference only[^1]. |
| 1.0 | 2019-10 | API stabilization; execution-provider model formalized. |
| 1.8 | 2021-06 | Web (WASM/WebGL) and mobile paths maturing. |
| 1.14 | 2023-02 | Training features folded into the main package. |
| 1.16 | 2023-09 | WebGPU EP work; expanded quantization tooling. |
| 1.17 | 2024-02 | DirectML and CUDA improvements; generative-AI split into onnxruntime-genai. |
| 1.20 | 2024-11 | Continued EP and mobile refinements[^2]. |

## References

[^1]: ONNX Runtime — project home and documentation. https://onnxruntime.ai
[^2]: ONNX Runtime releases (versions, dates, CUDA/cuDNN matrices). https://github.com/microsoft/onnxruntime/releases
[^3]: Execution Providers — architecture and supported backends. https://onnxruntime.ai/docs/execution-providers/

## Tags

cpp, machine-learning, inference, onnx, hardware-acceleration, model-serving, deep-learning, cross-platform, quantization, gpu
