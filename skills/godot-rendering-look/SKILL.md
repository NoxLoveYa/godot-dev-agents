---
name: godot-rendering-look
description: "Realistic look in Godot 4 Forward+: Environment (AgX, SDFGI, SSAO, volumetric fog, glow, exposure), custom sky shaders on a baked physical-atmosphere LUT with amortised volumetric clouds, water (refraction, depth absorption, ripple simulation, Snell window, caustics, underwater post), foliage (wind through global uniforms, backlight, alpha scissor / A2C), vegetation at scale (MultiMesh chunks, LOD, visibility ranges, shadow proxies), weather and seasons through global uniforms, and reference-driven iteration. Use for any shader, lighting or look task."
---

# Rendering and look development (Godot 4, Forward+)

Measured and verified in the reference project on Godot 4.7.2 (Forward+, d3d12, desktop GPU) unless marked
**(unverified)**. Performance numbers belong to that machine; re-measure on yours (`godot-performance`).

## Rules

1. **Reference first.** Every look task starts from a reference image (or a written checklist when there is none),
   its must-show list and numeric checks (`godot-cli-testing/references/metrics-and-references.md`). Accept with
   metrics + contact sheet + written verdict.
2. **One writer per field.** Environment, the key light, camera attributes and shared materials have an ownership
   table in the design; look code never writes another owner's field (e.g. enable flags that Settings owns).
3. **State in globals, look in one director.** Time, weather, season, wetness, snow and wind reach shaders as global
   shader uniforms (`[shader_globals]` with defaults equal to the default start); one director node turns state into
   engine fields through a look table keyed by sun elevation with weather modifiers.
4. **Amortise anything expensive that changes slowly** (clouds, sky radiance, sky captures, GI light updates): a few
   texels or one face per frame, at a lower rate, or only on change.
5. **Opaque where refraction looks through.** Screen-texture refraction only sees what was drawn before it: alpha
   scissor / hash, not alpha blend, for foliage, plants and particles seen through water.
6. **Quality scales by table.** Each expensive feature has a row per quality level, applied live; presets set the level.
7. **Iterate cheaply:** a sky-only or material-only shot scene and runtime override arguments before full-scene shots.

## Routing

| The task | Load |
|---|---|
| Environment, tonemap, exposure, GI, fog, glow, grade, key light | `references/environment-and-light.md` |
| Sky shader, atmosphere LUT, volumetric clouds, sun / moon / stars, sky capture | `references/sky-and-clouds.md` |
| Water surface, refraction, absorption, ripples, caustics, underwater | `references/water.md` |
| Foliage shaders, wind, translucency, vegetation at scale, LOD, shadow proxies | `references/foliage-and-vegetation.md` |
| Weather FX, wetness, snow, seasons, ice through globals | `references/weather-and-seasons.md` |
| The iteration loop against a reference | `workflows/look-dev-loop.md` |

## Inspiration and method sources

Schneider, "The Real-time Volumetric Cloudscapes of Horizon Zero Dawn" (SIGGRAPH 2015 / 2017, Guerrilla "Nubis");
Hillaire, "Physically Based Sky, Atmosphere and Cloud Rendering in Frostbite" (2016) and "A Scalable and Production
Ready Sky and Atmosphere Rendering Technique" (EGSR 2020); clayjohn's Godot demos (the amortised octahedral cloud map
idea of godot-volumetric-cloud-demo-v2, MIT). Use the ideas; write your own code.
