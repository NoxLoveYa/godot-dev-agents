# Timing a Godot frame from a script (windowed)

Frame time needs a real GPU: these are windowed runs. Run them only when the owner asked, one at a time, never next to
another agent's windowed run if you can avoid it.

## Reading GPU and CPU time

```gdscript
var rid := root.get_viewport_rid()
RenderingServer.viewport_set_measure_render_time(rid, true)
# each frame, after a warm-up:
var gpu_ms := RenderingServer.viewport_get_measured_render_time_gpu(rid)
var cpu_ms := RenderingServer.viewport_get_measured_render_time_cpu(rid)
```

Also useful: `Performance.get_monitor(Performance.TIME_PROCESS)`, `TIME_PHYSICS_PROCESS`, `RENDER_TOTAL_DRAW_CALLS_IN_FRAME`,
`RENDER_TOTAL_PRIMITIVES_IN_FRAME`, `RENDER_TOTAL_OBJECTS_IN_FRAME`; `Engine.get_frames_per_second()`. Run with vsync off
(the setting through a temp settings path, or `--disable-vsync`) and a fixed resolution (`--resolution 1920x1080`).

## Interleaved A/B in one process (the method that works on a shared GPU)

A before / after pair of separate runs is noise on a GPU other processes use. Instead, in one process:

1. Load the scene, place the camera, force and snap the state, wait ~3 s for caches and shader compilation.
2. Freeze anything that would differ between blocks for reasons other than the switch (stop the look director's
   process, freeze clocks).
3. Repeat PAIRS (e.g. 24) times: apply A, warm WARM frames (e.g. 20), average BLOCK frames (e.g. 40); apply B, the same.
4. Report per side the **minimum** and the **median** of the block means, and the deltas. Quiet blocks are the signal;
   bursts of other processes inflate the rest. The paired median of (A - B) is the robust delta.
5. Write one line per view into `out/perf_<tag>.txt`.

```
view=spawn         ON min 6.21 med 6.48 | OFF min 5.66 med 5.90 | delta min +0.55 med +0.58 ms
```

## Variants must remove things

Measure a part's cost by **removing** it: `visible = false`, a visibility range of 0, the feature's setting off, a
SubViewport set to `UPDATE_DISABLED`. Never by a property that changes what is drawn in another way:

- `cast_shadow = SHADOW_CASTING_SETTING_OFF` on a shadow-only proxy turns it into a **visible** object (a black occluder
  that hides geometry behind it), and the frame looked 5 ms cheaper for the wrong reason.
- Setting a material to unshaded changes overdraw and depth behaviour, not only lighting cost.

Print render-info counts (objects, primitives, draw calls, shadow casters) next to the times, so a variant that changed
more than intended shows.

## Bisecting a frame

Per-pass GPU timestamps had no names in the 4.7.2 build used, so the frame was taken apart by toggling things:
variants such as `noshadow`, `nosdfgi`, `noproxy`, `tree_lod<n>`, `fog0`, each an A/B against the full frame. Rank
the deltas; the top two or three explain most of the frame.

## Walk traces

Spikes come from movement (streaming, GI updates, LOD switches): a script that moves a free camera along a path at a
fixed speed for N frames and writes a CSV (frame, GPU ms, CPU ms, draw calls, primitives) plus a spike list.
`--speed=0` checks whether spikes happen standing still (SDFGI did); a `--nowarm` run shows first-use shader
compilation (only the first frame should be slow).
