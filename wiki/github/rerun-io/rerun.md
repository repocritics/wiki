# rerun-io/rerun

> Log, visualize, and query multimodal robotics/CV data on a time-aware columnar store — an SDK plus viewer, not just another plotting tool.

[GitHub repo](https://github.com/rerun-io/rerun) ·
[Official website](https://rerun.io) ·
[License: MIT OR Apache-2.0](https://github.com/rerun-io/rerun/blob/main/LICENSE-APACHE)

## Overview

Rerun is an open-source SDK (Python, Rust, C++) and viewer for logging and visualizing multimodal, time-varying data: images, point clouds, tensors, 3D transforms, joint states, video, and time series. It was built by Rerun Technologies AB (Stockholm) and open-sourced in February 2023[^1]; the viewer is written in Rust on top of egui and wgpu — Rerun's CTO Emil Ernerfeldt is the author of egui[^2]. The pitch has shifted over time from "visual debugger for computer vision" to "the data layer for physical AI": the same logged data is now queryable as dataframes or SQL and streamable into training pipelines[^3].

The core design bet is that visualization and data management are the same problem. Instead of rendering the latest frame like RViz, Rerun ingests everything into an in-memory, Arrow-based, time-indexed database, so you can scrub backwards through a robot run, compare sensors side-by-side at any timestamp, and later query the same recording for dataset extraction. That bet gives it capabilities pure visualizers lack, at the cost of RAM: the viewer holds history in memory and must garbage-collect under a memory limit[^4].

It is still pre-1.0 after three-plus years (11.1k stars, active daily development as of mid-2026); the maintainers explicitly warn to expect breaking changes[^5], and frequent 0.x minors with migration steps are the main tax on adopters. The business is open-core: everything in this repo is free, while "Rerun Hub", a hosted catalog for robotics data, is the commercial product[^6].

## Getting Started

```bash
pip install rerun-sdk   # SDK + bundled viewer (Python only)
# Rust: cargo add rerun    C++: see rerun_cpp/README.md
# standalone viewer: cargo install rerun-cli --locked --features nasm
```

```python
import rerun as rr
import numpy as np

rr.init("my_app")
rr.spawn()                          # launch a viewer and connect to it
# rr.save("run.rrd")                # ...or stream to a file instead

rr.set_time("frame", sequence=42)   # all subsequent logs tagged frame=42

positions = np.random.rand(100, 3)
rr.log("world/points", rr.Points3D(positions, radii=0.02))
rr.log("world/camera/image", rr.Image(np.zeros((480, 640, 3), dtype=np.uint8)))
```

Entity paths (`world/camera/image`) form a hierarchy; logging a `Transform3D` at a path positions everything beneath it in 3D space.

## Architecture / How It Works

The repo is a large Cargo workspace of `re_*` crates[^7]. The main pieces:

- **Data model** — ECS-inspired. You log *archetypes* (`Points3D`, `Image`, `Transform3D`, ...) to *entity paths*; archetypes decompose into typed *components*. Type definitions live in an IDL and code-generation emits the Python, Rust, and C++ APIs in lockstep, which is why the three SDKs stay near-identical.
- **Storage** — everything becomes Apache Arrow columns. Since 0.18 (2024) the store is chunk-based (`re_chunk_store`): logs are batched into column chunks, which cut per-row overhead and made the later dataframe/SQL query APIs feasible[^8]. Recordings serialize to `.rrd` files.
- **Timelines** — data is indexed on multiple timelines at once (wall-clock `log_time`, frame counters, custom clocks). Static data (e.g. a mesh) is logged once and joined against every timestamp. Time-scrubbing is a query against this index, not a replay.
- **Viewer** — egui + wgpu with a custom renderer (`re_renderer`). Compiles to WASM, so the same viewer runs natively and in the browser. *Blueprints* — the view layout and per-view settings — are themselves data stored in the same store, and can be constructed from code in Python.
- **Transport** — SDKs batch chunks on a background thread and ship them over gRPC to a viewer or proxy (the original raw-TCP protocol was replaced by gRPC in the 0.22/0.23 era, a breaking transport change). Ingestion also accepts third-party formats, notably MCAP and LeRobot datasets[^9].

The coupling story: SDK, wire format, store, and viewer co-evolve inside one repo and one release train. That keeps the three language SDKs consistent, but it means a viewer from one 0.x version cannot be assumed to read data from another — SDK, viewer, and `.rrd` files effectively version together.

## Production Notes

- **Memory is the operating constraint.** The viewer keeps history in RAM. Long-running or high-rate streams need `--memory-limit`, which evicts the oldest data (turning Rerun into a rolling buffer, RViz-style)[^4]. Budget for the viewer's RAM the way you budget for a database, not a GUI.
- **Known scaling cliffs, documented by the maintainers themselves:** the viewer slows down with very many entities[^10] and multi-million-point clouds can be slow[^11]. Many small entities are worse than few large batched ones; prefer batch archetypes over per-object paths.
- **0.x churn is real.** APIs get renamed (e.g. the time API consolidation into `rr.set_time`), transports get replaced, and `.rrd` files have historically not been guaranteed loadable across versions. Pin exact SDK versions per project and treat `.rrd` as a working format, not an archival one — export what you need via the dataframe API.
- **Install asymmetry.** Only the Python wheel bundles the viewer; Rust and C++ users need a separate `rerun-cli` install, which itself wants the `nasm` feature (and the nasm assembler) for acceptable video decoding[^5].
- **ROS is an integration, not a native citizen.** No first-class ROS transport in core; teams bridge via examples or MCAP ingestion[^9].
- **Logging overhead in hot loops is mostly hidden** by background batching, but serializing large raw images per-frame in Python still costs; prefer logging encoded video streams over raw frames.

## When to Use / When Not

**Use when:**
- You debug robotics/CV/SLAM pipelines and need synchronized scrubbing across camera, depth, lidar, poses, and time series.
- You want one logging call-site to serve visualization *and* later dataset extraction (dataframe/SQL queries over recordings).
- You need the same viewer on native and web (WASM) without separate builds.
- You work across Python, Rust, and C++ and want API-consistent SDKs.

**Avoid when:**
- You need a stable, archival data format today — 0.x compatibility churn makes `.rrd` risky as a system of record.
- You are all-in on ROS and only need topic introspection — RViz or Foxglove are lower-friction there.
- Your workload is billions of points or tens of thousands of entities per scene — you will hit the documented scaling cliffs[^10][^11].
- You just need 2D charts or simple time series — this is heavy machinery for that job.

## Alternatives

- foxglove/foxglove — ROS/MCAP-native robotics observability; note the studio app went closed-source in 2024. Use it when your world is ROS topics and you want a supported commercial product.
- ros-visualization/rviz — the ROS default; latest-state visualization only, no history/query. Use it for live ROS introspection with zero new deps.
- facontidavide/PlotJuggler — time-series-first plotting for robotics logs. Use it when your problem is signals, not 3D scenes.
- nerfstudio-project/viser — Python web-based 3D visualization library. Use it when you want scriptable 3D scenes without a logging/storage layer.
- isl-org/Open3D — 3D data processing library with a visualizer. Use it when you need geometry algorithms, not temporal logging.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2 | 2023-02 | First public release; open-sourcing announcement[^1]. |
| 0.10 | 2023-10 | C++ SDK joins Python and Rust. |
| 0.15 | 2024-04 | Blueprints (view layouts) configurable from Python code. |
| 0.18 | 2024-08 | Chunk-based store rework; large memory-overhead reduction[^8]. |
| 0.19 | 2024-10 | Dataframe query API; video asset support. |
| 0.22 | 2025-02 | Transport migration toward gRPC begins; TCP deprecated. |
| 0.2x | 2025–2026 | MCAP/LeRobot ingestion, SQL queries, "data layer for physical AI" repositioning[^3]. Still pre-1.0. |

## References

[^1]: Rerun, "What is Rerun?" — docs overview. https://rerun.io/docs/overview/what-is-rerun
[^2]: emilk/egui — immediate-mode GUI library underlying the viewer. https://github.com/emilk/egui
[^3]: rerun-io/rerun README, "The data layer for physical AI" + dataframe/SQL docs. https://rerun.io/docs/howto/query-and-transform/get-data-out
[^4]: Rerun docs, "How to limit memory use". https://rerun.io/docs/howto/visualization/limit-ram
[^5]: rerun-io/rerun README, "Status" section ("Expect breaking changes!"). https://github.com/rerun-io/rerun#status
[^6]: Rerun pricing — open-core model, Rerun Hub. https://rerun.io/pricing
[^7]: ARCHITECTURE.md — crate map and internals. https://github.com/rerun-io/rerun/blob/main/ARCHITECTURE.md
[^8]: Rerun release notes (0.18 chunk store and later). https://github.com/rerun-io/rerun/releases
[^9]: Rerun docs, "MCAP ingestion". https://rerun.io/docs/howto/logging-and-ingestion/mcap
[^10]: rerun-io/rerun#7115 — viewer slows down with too many entities. https://github.com/rerun-io/rerun/issues/7115
[^11]: rerun-io/rerun#1136 — multi-million point clouds can be slow. https://github.com/rerun-io/rerun/issues/1136

## Tags

rust, python, cpp, robotics, computer-vision, visualization, multimodal-data, data-logging, apache-arrow, sdk, 3d, time-series
