---
name: godot-performance
description: "Godot 4 performance: budgets per feature and preset, profiling (Visual Profiler, RenderingServer GPU timings, render info), interleaved A/B on a shared GPU, measured Forward+ lessons (LOD fade margins, SDFGI cascades, shadow gating, sky radiance, shadow proxies), quality presets and live scaling, CPU throttling of AI, allocation-free per-frame code. Use when a frame is slow, a budget is set, or a quality preset is designed."
---

# Performance (Godot 4)

## Rules

1. **Budgets in the design**, per feature, in GPU ms and CPU ms on the reference machine and preset (e.g. "+2.5 ms GPU,
   +1.0 ms CPU for the whole atmosphere at 1080p High; High ≥ 60 fps in every acceptance view"). A baseline is recorded
   before work starts; each package reports its delta against it for the same views.
2. **Measured only when the owner asks** (or the brief carries an owner-approved budget). Frame time needs windowed
   runs: they can steal focus (`godot-cli-testing`).
3. **Interleaved A/B in one process**, min and median of block means, never a before / after pair of separate runs on
   a shared GPU (`godot-cli-testing/references/perf-ab.md`).
4. **Variants remove things** (`visible = false`, range 0, setting off). Changing `cast_shadow` on a shadow-only proxy
   makes it visible and fakes a 5 ms win.
5. **Every expensive feature has a quality row** applied live, and presets set the rows.
6. **Per-frame code allocates nothing** and heavy thinking runs at a lower, staggered rate.

## Routing

| The task | Load |
|---|---|
| Find what the frame pays for | `references/profiling.md`, then `workflows/performance-audit.md` |
| Known costs and fixes in Forward+ (forest, sky, shadows, GI, water, particles) | `references/measured-lessons.md` |
| Design or extend quality presets and live scaling | `references/quality-presets.md` |
| CPU: AI, scripts, allocation, physics queries | `references/cpu-patterns.md` |
| Timing scripts and the A/B harness | `godot-cli-testing/references/perf-ab.md` |
