# triton-inference-server/server

> NVIDIA's model-serving runtime that fronts many inference backends (TensorRT, PyTorch, ONNX, vLLM, Python) behind one HTTP/gRPC API.

[GitHub repo](https://github.com/triton-inference-server/server) ·
[Official docs](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html) ·
[License: BSD-3-Clause](https://github.com/triton-inference-server/server/blob/main/LICENSE)

## Overview

Triton Inference Server is NVIDIA's open-source inference-serving layer, first released in 2018 as "TensorRT Inference Server" and renamed to Triton in 2019 to reflect that it serves far more than TensorRT[^1]. Its job is narrow and load-bearing: take a directory of trained models, expose them over standardized HTTP/REST and gRPC endpoints, and squeeze GPU utilization through batching and concurrent execution. It does not train models and it does not, by itself, implement any framework's math — it delegates that to pluggable *backends*.

The intended user is a team that has already trained models (often several, in different frameworks) and needs to serve them in production on NVIDIA GPUs with predictable latency and high throughput. Triton's defining bet is the model repository plus backend abstraction: one server process can host a TensorRT engine, a PyTorch TorchScript model, an ONNX graph, and an arbitrary Python function simultaneously, each with its own scheduling and batching policy, and route between them via ensembles or Business Logic Scripting[^2].

The central tension is coupling to the NVIDIA stack. Triton runs on CPU and ARM, but its reason to exist is GPU serving, and the supported, tested distribution is the monthly NGC container tagged to a CUDA/driver matrix. You gain a mature, batteries-included serving runtime; you accept that the happy path is Docker images from NVIDIA GPU Cloud and a release cadence tied to NVIDIA's container calendar rather than semantic versioning of the repo alone.

## Getting Started

Triton is distributed as prebuilt NGC containers; building from source is possible but not the recommended path.

```bash
# Fetch an example model repository
git clone -b r26.06 https://github.com/triton-inference-server/server.git
cd server/docs/examples
./fetch_models.sh

# Launch the server, loading one ONNX model explicitly
docker run --gpus=1 --rm --net=host \
  -v ${PWD}/model_repository:/models \
  nvcr.io/nvidia/tritonserver:26.06-py3 \
  tritonserver --model-repository=/models \
    --model-control-mode explicit --load-model densenet_onnx
```

A model repository is a directory convention, not a config file. Each model is a subdirectory with numbered version folders and an optional `config.pbtxt`:

```
model_repository/
  densenet_onnx/
    config.pbtxt        # platform, inputs/outputs, batching, instance_group
    1/
      model.onnx
```

Inference then goes over the KServe v2 protocol on port 8000 (HTTP) / 8001 (gRPC); port 8002 serves Prometheus metrics.

## Architecture / How It Works

Triton is a C++ core (`tritonserver` / the `tritonserver` library) that owns model lifecycle, request scheduling, and the network frontends, with everything framework-specific pushed into backends loaded as shared libraries[^2].

- **Backends.** Each backend is a `.so` implementing the Triton Backend API (`TRITONBACKEND_*`). TensorRT, ONNX Runtime, PyTorch, OpenVINO, TensorFlow, and Python are the common ones; the vLLM and TensorRT-LLM backends serve LLMs. The core has no knowledge of tensors' meaning — it hands the backend an input tensor and gets an output tensor back. Custom backends (C/C++ or Python-based) plug in the same way.
- **Model repository & control modes.** Models are discovered from one or more repositories. `--model-control-mode` is `none` (load all at startup), `explicit` (load/unload via API), or `poll` (watch the filesystem). Explicit mode is standard in production so deployments are deterministic.
- **Schedulers and batching.** The differentiating internals. The *dynamic batcher* holds incoming requests for a configurable `max_queue_delay_microseconds` to coalesce them into a larger batch, trading a little latency for throughput. The *sequence batcher* pins a stateful sequence of requests to one model instance and supports implicit state management for recurrent/stateful models. `instance_group` controls how many concurrent copies of a model run and on which GPUs — this is how Triton fills a GPU with concurrent model execution.
- **Ensembles and BLS.** An *ensemble* wires models into a static DAG (preprocess → infer → postprocess) executed inside the server with no client round-trips. *Business Logic Scripting* is the escape hatch: a Python backend model that calls other models imperatively, enabling loops and conditionals an ensemble can't express[^2].
- **Frontends & in-process APIs.** HTTP/gRPC implement the community KServe predict-v2 protocol[^3]. A C API and Java API let you link the server library directly into a host process for edge / in-process serving without the network hop.

The coupling story: the repo you are reading (`server`) is the core plus frontends. The backends, client libraries, `model_analyzer`, and `perf_analyzer` each live in their own repositories under the same org, and are assembled into the shipped container. Reading this repo alone under-represents the system; most operational behavior lives in the backend repos and the `config.pbtxt` schema.

## Production Notes

- **The container is the unit of truth.** Version tags like `26.06-py3` encode the NGC container release, which pins CUDA, cuDNN, and driver requirements. The Git tag `r26.06` and server version (2.70.0 corresponds to 26.06) are a separate axis. Mismatching a container against the host driver/CUDA is the most common first-day failure. Upgrades mean adopting a new container and re-validating the driver matrix, not bumping a package version.
- **Batching config is the performance lever, and it is per-model.** Default configs do not enable dynamic batching. Latency-vs-throughput is governed by `max_queue_delay_microseconds`, `preferred_batch_size`, and `instance_group` count — all in `config.pbtxt`. Getting these wrong is the difference between a saturated GPU and an idle one. Use `perf_analyzer` and `model_analyzer` (separate repos) to sweep the space rather than guessing.
- **Shared-memory transport for large tensors.** For image/video/embedding payloads, HTTP/gRPC serialization dominates latency. Triton supports CUDA and system shared-memory so clients can hand off tensors without copying them through the network stack — required for high-throughput vision pipelines but adds client-side complexity.
- **LLM serving is a moving target.** vLLM and TensorRT-LLM backends bring Triton into LLM serving, but this space evolves fast and the ergonomics lag standalone servers. Teams frequently find raw vLLM or TGI simpler for a single LLM; Triton earns its place when the LLM sits alongside other models under one serving fabric. NVIDIA's newer NIM microservices wrap Triton for this use case.
- **Memory and model loading.** Concurrent instances multiply GPU memory use; loading many large models can exhaust VRAM silently until a load fails. Explicit control mode plus staged loading is the safe pattern.
- **Open issue volume is high** (~900 open), typical for a project spanning many backends and hardware permutations — many are backend- or version-specific rather than core defects. Metrics on port 8002 (Prometheus) and per-model stats are essential for diagnosing which backend a regression lives in.

## When to Use / When Not

**Use when:**
- You serve multiple models, possibly across frameworks, and want one runtime, one API, and one metrics surface.
- You need GPU throughput via dynamic/sequence batching and concurrent execution, and you are on NVIDIA hardware.
- You want in-server pipelines (ensembles/BLS) so preprocessing and postprocessing run without client round-trips.
- You are already on the NVIDIA stack (TensorRT, NGC, AI Enterprise support).

**Avoid when:**
- You serve exactly one model and want the simplest possible deploy — a framework-native server (TorchServe, ONNX Runtime server, plain FastAPI) is less machinery.
- You are serving a single LLM and want the current best ergonomics — vLLM or TGI directly is usually simpler.
- You are not on NVIDIA GPUs and don't need GPU features; the CPU path works but you're carrying weight you won't use.
- You need lockstep semantic versioning independent of NVIDIA's container release calendar.

## Alternatives

- vllm-project/vllm — use instead when you serve one or a few LLMs and want state-of-the-art token throughput and paged-attention ergonomics without Triton's config surface.
- pytorch/serve (TorchServe) — use instead when you're PyTorch-only and want a lighter, framework-native server.
- kserve/kserve — use instead when you want Kubernetes-native model serving with autoscaling and canaries; KServe can actually run Triton as a backing runtime.
- bentoml/BentoML — use instead when you want to package Python inference services with business logic and flexible deployment targets over raw GPU batching.
- ray-project/ray (Ray Serve) — use instead when serving is one part of a larger distributed Python application and you want programmatic composition over a config-driven repository.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-10 | Initial release as TensorRT Inference Server[^1]. |
| — | 2019 | Renamed to Triton Inference Server; backend abstraction generalized beyond TensorRT[^1]. |
| 2.0 | 2020 | Rearchitected around the pluggable Backend API and KServe v2 protocol[^3]. |
| — | 2023 | vLLM and TensorRT-LLM backends added for LLM serving. |
| 2.70.0 | 2026-06 | Current release; ships as NGC container 26.06[^4]. |

## References

[^1]: NVIDIA Developer, "Triton Inference Server" product page (history of the TensorRT Inference Server rename). https://developer.nvidia.com/nvidia-triton-inference-server
[^2]: Triton architecture documentation — backends, schedulers, ensembles, and Business Logic Scripting. https://github.com/triton-inference-server/server/blob/main/docs/user_guide/architecture.md
[^3]: KServe predict v2 (community inference protocol implemented by Triton's HTTP/gRPC frontends). https://github.com/kserve/kserve/tree/master/docs/predict-api/v2
[^4]: Triton Inference Server releases (2.70.0 / NGC 26.06). https://github.com/triton-inference-server/server/releases/latest

## Tags

inference-server, machine-learning, deep-learning, gpu, nvidia, model-serving, tensorrt, cpp, python, mlops, llm-serving
