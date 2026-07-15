# Zulko/moviepy

> Python scripting layer over FFmpeg for programmatic, non-linear video editing — cuts, composites, and per-frame effects expressed as numpy arrays.

[GitHub repo](https://github.com/Zulko/moviepy) ·
[Official website](https://zulko.github.io/moviepy/) ·
[License: MIT](https://github.com/Zulko/moviepy/blob/master/LICENSE.txt)

## Overview

MoviePy is a Python library for video editing: trimming, concatenation, title
insertion, compositing, and custom effects. It was written by Zulko and first
released in 2013[^1]. Its core idea is that a video is a Python object whose
frames are numpy arrays, so an effect is just a function `frame -> frame` and a
timeline is just an object graph of clips. That model makes it approachable for
people who already think in numpy and want scripted, reproducible edits rather
than a GUI timeline.

The library does not decode or encode video itself. It shells out to FFmpeg
(vendored through `imageio-ffmpeg`) for reading and writing, and uses Pillow and
numpy for the in-Python frame manipulation in between. This is the defining
tradeoff: MoviePy is flexible and readable, but every frame is round-tripped
through Python, which makes it materially slower than an equivalent raw FFmpeg
filtergraph — the README states this outright[^2]. For 30-second title cards it
is irrelevant; for hour-long 4K renders it dominates wall-clock time.

The other defining fact is the **v2.0 rewrite**, which shipped a large set of
breaking changes and is not source-compatible with the v1.x code that most
tutorials and Stack Overflow answers still assume[^3]. Anyone landing on MoviePy
today has to first figure out which major version a given example targets.

## Getting Started

```bash
pip install moviepy      # pulls imageio-ffmpeg, which vendors an FFmpeg binary
```

```python
# v2 API. Note the `with_*` / `subclipped` naming — this is NOT v1.
from moviepy import VideoFileClip, TextClip, CompositeVideoClip

clip = (
    VideoFileClip("example.mp4")
    .subclipped(10, 20)               # keep 00:10–00:20
    .with_volume_scaled(0.8)          # audio to 80%
)

txt = (
    TextClip(font="Arial.ttf", text="Hello there!", font_size=70, color="white")
    .with_duration(10)
    .with_position("center")
)

CompositeVideoClip([clip, txt]).write_videofile("result.mp4")
```

Clips are immutable-ish: `with_*` methods return a new clip rather than mutating
in place, so pipelines chain cleanly. `write_videofile` is where FFmpeg is
invoked and where most runtime and most errors happen.

## Architecture / How It Works

A `Clip` exposes `get_frame(t)` returning the frame at time `t`. Everything else
is built on that:

- **Readers.** `VideoFileClip` spawns an FFmpeg subprocess and reads raw RGB
  frames off a pipe on demand; `AudioFileClip` does the same for PCM samples.
  Seeking is approximate — FFmpeg is asked to jump near a timestamp, so
  frame-exact seeks on long-GOP codecs can be off by a frame.
- **Compositing.** `CompositeVideoClip` layers clips by evaluating each layer's
  `get_frame(t)` and its position/mask, then alpha-blends in numpy. `concatenate_videoclips`
  and `clips_array` are the timeline primitives.
- **Effects.** In v2, effects are classes implementing an `apply(clip)` contract,
  applied via `clip.with_effects([...])`. In v1 they were `clip.fx(func, ...)`
  calls plus the `vfx`/`afx` function modules — a rename that breaks nearly every
  old snippet[^3].
- **Text.** `TextClip` renders through Pillow in v2. In v1 it shelled out to
  **ImageMagick**, whose install/policy configuration was the single most common
  source of "works on my machine" failures[^4]. Dropping that dependency is one
  of v2's more consequential changes.
- **Writers.** `write_videofile` pipes the numpy frames back into an FFmpeg
  subprocess for encoding. Codec, bitrate, and preset are passed straight through
  to FFmpeg flags.

The whole library is a thin, legible orchestration layer. There is no internal
video pipeline, no GPU path, and no async/streaming model — clips are pulled
frame by frame, in Python, single-threaded per render.

## Production Notes

- **Throughput is the constraint.** Because frames cross the Python boundary
  twice (decode pipe in, encode pipe out) with numpy work in between, MoviePy is
  a poor fit for high-volume or real-time transcoding. If your edit is
  expressible as an FFmpeg filtergraph (crop, scale, overlay, concat), doing it
  in FFmpeg directly is often an order of magnitude faster. Reach for MoviePy
  when the per-frame logic genuinely needs Python (data-driven overlays,
  numpy/ML-generated frames).
- **FFmpeg is load-bearing and mostly invisible.** The vendored `imageio-ffmpeg`
  binary is convenient but may lag codec support or lack hardware encoders
  (NVENC, VAAPI). Production setups frequently point MoviePy at a system FFmpeg
  via `IMAGEIO_FFMPEG_EXE` to get the codecs and acceleration they need.
- **Subprocess and temp-file hygiene.** Renders spawn FFmpeg processes and write
  temporary audio/video files. Long-running services should ensure clips are
  closed (`clip.close()` or context managers) to release file handles and reap
  subprocesses; leaks here are a recurring source of "too many open files".
- **Memory.** Frames are full numpy arrays. High-resolution composites with many
  layers can spike RAM, and masks double the footprint. There is no built-in
  frame cache eviction policy you can rely on for very long clips.
- **v1 → v2 migration is not mechanical.** Import path (`from moviepy.editor
  import *` is gone — it is now `from moviepy import ...`), method renames
  (`subclip`→`subclipped`, `set_position`→`with_position`, `volumex`→
  `with_volume_scaled`), and the effects API all changed. Pin `moviepy<2` if you
  cannot migrate immediately; the v1 docs remain hosted but v1 is unmaintained[^2].
- **Maintenance bandwidth is explicitly limited.** The README carries a standing
  "maintainers wanted" notice; the project has real activity but is small-team
  and volunteer-run, so expect slower turnaround on niche issues[^2].

## When to Use / When Not

**Use when:**
- Your edit needs real Python per frame: data-driven text, plots, generated or
  ML-produced frames, procedural animation.
- You want readable, version-controllable, reproducible edit scripts instead of a
  GUI.
- Volume and latency are modest (title cards, GIFs, short social clips, batch
  jobs where minutes-per-video is acceptable).

**Avoid when:**
- The transformation is a plain FFmpeg operation (transcode, crop, scale,
  concat, overlay) — use FFmpeg or a thin binding and skip the Python round-trip.
- You need real-time, streaming, or GPU-accelerated pipelines.
- You need frame-exact seeking guarantees or precise A/V sync on difficult
  sources — the subprocess/pipe model makes this fiddly.

## Alternatives

- kkroening/ffmpeg-python — build FFmpeg filtergraphs in Python; far faster since
  frames never enter Python. Use when your edit is expressible as FFmpeg filters.
- PyAV-Org/PyAV — Pythonic bindings to FFmpeg's libav* libraries; lower-level,
  much faster decode/encode. Use when you need frame access with performance.
- abhiTronix/vidgear — high-throughput video I/O framework with hardware and
  streaming support. Use for real-time or high-volume capture/processing.
- opencv/opencv — cv2 gives per-frame numpy access like MoviePy but no audio,
  compositing, or clean encoding story. Use for CV/vision, not editing.
- imageio/imageio — simple frame read/write over FFmpeg with no editing layer.
  Use when you only need to get frames in and out.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013-08 | First public release by Zulko; numpy-frame + FFmpeg model[^1]. |
| 1.0.0 | 2020-11 | API stabilized; `moviepy.editor` convenience import; TextClip via ImageMagick[^4]. |
| 1.0.3 | 2021 | Final maintained v1.x line; still referenced by most tutorials[^2]. |
| 2.0.0 | 2024 | Breaking rewrite: `with_*`/`subclipped` naming, effects-as-classes, ImageMagick dropped for Pillow, `moviepy.editor` removed[^3]. |

## References

[^1]: MoviePy — repository created 2013-08-12, written by Zulko, MIT-licensed. https://github.com/Zulko/moviepy
[^2]: MoviePy README — "slower than using ffmpeg directly," Python 3.9+, v1 unmaintained, maintainers-wanted notice. https://github.com/Zulko/moviepy/blob/master/README.md
[^3]: MoviePy docs, "Updating from v1.0 to v2.0." https://zulko.github.io/moviepy/getting_started/updating_to_v2.html
[^4]: MoviePy v1.0.3 documentation (archived). https://zulko.github.io/moviepy/v1.0.3/

## Tags

python, video-editing, video-processing, ffmpeg, numpy, gif, compositing, media, scripting, animation
