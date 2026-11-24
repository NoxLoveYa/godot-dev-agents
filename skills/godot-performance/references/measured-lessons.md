# Measured lessons (Godot 4.7.2, Forward+, d3d12, 1080p, a desktop GPU shared with other apps)

Each number came from interleaved A/B in one process. Re-measure on your project; the direction of each lesson
carries over, the milliseconds do not.

| Topic | Finding | Fix |
|---|---|---|
| LOD fade margins | With `VISIBILITY_RANGE_FADE_SELF`, two LODs draw across each margin. 7 / 16 / 24 m margins on trees = ~41k triangles of overlapping alpha-tested cards per tree in the 16-47 m ring: **5-9 ms** of the High frame in the forest | margins x 0.3 on High (cross-fade over 30% of the margin), full margins only on Ultra; or fade off on dense layers |
| SDFGI cascades | each cascade ~0.5 ms average and most of the 3-6 ms spikes, also standing still; cascades 3-4 only lit the fogged far forest | 4 -> 2 cascades on Low-High (picture change < 3 levels in the far forest); 4 on Ultra |
| SDFGI light updates | a moving sun re-lights GI | `frames_to_update_lights` 4 (8 on Low); change `sdfgi_energy` only on big moves |
| Directional shadow in storms | the shadow map was ~a quarter of the High frame in a thunderstorm while shadows were invisible | gate `shadow_enabled` off when key energy x opacity < a threshold, with an opacity fade and hysteresis |
| Shadow of far tree cards | alpha-tested leaf cards 55-145 m away cost 3.5-4.5 ms of shadow fill at a low sun | shadow-only proxies per band; closed crown hulls beyond ~55 m |
| Low-sun cost under water | +3 ms at a low sun was the engine's sun shadow pass, not the custom shaft march | measure before optimising the obvious suspect |
| Measurement artefact | `cast_shadow = OFF` on a shadows-only proxy made it a visible black occluder, the frame looked 5 ms cheaper | variants hide (`visible = false`) |
| Sky radiance | REALTIME ~0.35 ms; INCREMENTAL ~0.05 ms; a sky shader using `TIME` re-renders radiance every frame | INCREMENTAL, no `TIME` in the sky shader, animate uniforms from script |
| Volumetric clouds | full-res march per pixel is unaffordable; an octahedral map marched 1/16 per frame cost +0.14 / +0.18 / +0.35 ms (Low / Medium / High) | amortise; the sky main pass only samples the map |
| Sky in the full scene | +0.5-0.6 ms flat, mostly ambient / reflection sampling and SDFGI sky light, not the clouds | budget it as the cost of a dynamic sky |
| Sky capture for water | 512x256 at 10 Hz ~0.02 ms average | `UPDATE_ONCE` on a timer, rate by quality |
| Live quality change | re-instancing a module scene took ~45 ms; caching spot lists across it took it to ~4 ms | hand caches to the new instance |
| Dense aquatic beds | alpha-scissor overdraw under water is the worst view | clumps, 40 m range, wave LOD by distance, density by setting |
| Rain | 12k drops + splashes + heightfield ~0.45 ms GPU | counts by quality (3k / 6k / 12k), splashes off on Low |
| Ambience mixer | ~21 µs per frame | budget 100 µs; headless probe |
| First frame | shader compilation makes only frame 0 slow after a cold start | ignore in walk traces (`--nowarm` run shows it); warm-up before timing |

## Budget table shape (design doc)

| Feature | GPU ms | CPU ms | Notes |
|---|---|---|---|
| Sky main + cloud map + radiance + capture | 0.14 / 0.18 / 0.35 | — | per quality, measured method named |
| SDFGI light updates with a moving sun | 0.30 | — | frames_to_update_lights 4 |
| Rain 12k + splashes + heightfield | 0.45 | 0.05 | storm only |
| Fauna | 0.15 | 0.30 | |
| **Worst case** | **≈ 2.4** | **≈ 0.9** | exclusive features not summed |
