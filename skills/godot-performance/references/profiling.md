# Profiling a Godot 4 frame

## Tools

| Tool | What it gives | Notes |
|---|---|---|
| Editor Debugger > Profiler | script function time per frame | run the game from the editor; human-driven |
| Editor Debugger > Visual Profiler | CPU / GPU time per render pass | names depend on the build; human-driven |
| `RenderingServer.viewport_get_measured_render_time_gpu/cpu(rid)` | GPU / CPU ms of a viewport | enable with `viewport_set_measure_render_time(rid, true)`; scriptable |
| `Performance.get_monitor(...)` | `TIME_FPS`, `TIME_PROCESS`, `TIME_PHYSICS_PROCESS`, `RENDER_TOTAL_DRAW_CALLS_IN_FRAME`, `RENDER_TOTAL_PRIMITIVES_IN_FRAME`, `RENDER_TOTAL_OBJECTS_IN_FRAME`, `RENDER_VIDEO_MEM_USED`, memory and object counts | scriptable, cheap |
| `RenderingServer.get_rendering_info(...)` / `viewport_get_render_info(rid, type, info)` | per-viewport objects / primitives / draw calls, including the shadow pass type | compare variants: a variant that changes counts unexpectedly changed more than intended |
| `Time.get_ticks_usec()` around a function | CPU cost of one call | average over many calls in a headless probe |
| Walk trace | per-frame CSV along a camera path | spikes from movement, LOD switches, GI, streaming |

In the 4.7.2 build used, per-pass GPU timestamps had no pass names when read from script, so frames were taken apart
by toggling features (bisect A/B), which is reliable and scriptable.

## Method

1. Fix the conditions: resolution, preset, vsync off, the named views, the world state snapped and frozen, a warm-up
   (~3 s, plus a no-warm run to see shader compilation once).
2. Record the frame per view: GPU ms (min / median of blocks), CPU ms, draw calls, primitives, objects.
3. Bisect with A/B variants that remove one feature each; rank the deltas.
4. For spikes, a walk trace: is it there standing still? With the feature off?
5. For CPU, a headless probe that times the suspect `_process` over many frames (e.g. an ambience mixer ~21 µs per
   frame against a 100 µs budget; a fish tick with and without ice).
6. Write the numbers with their method into the design doc's budget table; change, re-measure the same way.

## What to suspect first in Forward+ outdoor scenes

1. Shadow pass: casters (alpha-tested cards), cascade count and distance, atlas size, a low sun (long shadows, more
   casters in the frustum).
2. Overdraw of alpha-tested vegetation, doubled by LOD fade margins.
3. SDFGI: cascades, light updates with a moving sun.
4. Volumetric fog volume size and depth.
5. Sky: radiance updates (REALTIME or a sky shader using `TIME`), sky captures, cloud marching.
6. Screen-space passes: SSR, SSAO, SSIL quality, glow upscale mode.
7. Custom post / water passes: march step counts.
