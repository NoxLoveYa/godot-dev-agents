# Quality presets and live scaling

## Shape

- Settings keys for the big switches (render scale, upscaler, TAA, shadows level, GI, SSAO, SSR, volumetric fog level,
  vegetation density, view distance, an "atmosphere / effects quality" 0-2) and presets LOW / MEDIUM / HIGH / ULTRA /
  CUSTOM that set them (`godot-game/references/saving-and-settings.md`).
- Each module reads its quality key and maps it through its own table (constants at the top of the owning script), so
  a preset never needs to know module internals:

| Feature | 0 (Low) | 1 (Medium) | 2 (High / Ultra) |
|---|---|---|---|
| Cloud map / march steps / light samples | 320² / 40 / 3, no detail | 640² / 52 / 4 | 1024² / 88 / 6 |
| Sky radiance (INCREMENTAL) | 128 | 256 | 256 |
| Sky capture | 256x128 at 4 Hz | 512x256 at 10 Hz | 512x256 at 15 Hz |
| SDFGI light update frames | 8 | 4 | 4 |
| Water SSR / underside / sky taps / caustic channels / shaft steps | 16 / 0 / 1 / 1 / 6 | 32 / 12 / 5 / 3 / 14 | 48 / 32 / 5 / 3 / 28 |
| Rain drops / splash cap | 3000 / 0 | 6000 / 300 | 12000 / 800 |
| Snow flakes | 2500 | 5000 | 9000 |
| Fauna counts | low | medium | high |
| Plant wave LOD | lean only beyond 15 m | wave to 25 m | wave to 40 m |

Engine-level rows Settings applies itself:

| Key | Levels |
|---|---|
| shadows | atlas 1024 / 2048 / 4096 / 4096, cascade mode, soft filter quality 1-4 |
| volumetric fog | off / 32x32 / 64x64 volume |
| vegetation density | 0.3 / 0.65 / 1.0 of instances on named layers |
| view distance | visibility ranges x 0.55 / 0.8 / 1.0 / 1.3 |
| tree LOD margin scale | 1.0 / 1.0 / 0.3 / 1.0 (High trades cross-fade for 5-9 ms) |
| SDFGI cascades | 2 / 2 / 2 / 4 |

## Live apply

- A setting change applies at once (no restart) for everything that can: Environment flags, shadow atlas, light
  settings, visibility ranges, instance counts (`MultiMesh.visible_instance_count`), particle `amount_ratio`, viewport
  sizes, shader uniforms.
- Things built once at startup (a fauna module that reads quality in `_ready`) are re-instanced by their loader,
  deferred so several changes in one frame rebuild once, with expensive caches handed over.
- Project settings read once at startup (some rendering settings) cannot be live: keep them out of the presets or
  document "after restart".
- Test the live change headless (the preset sets the keys, modules see the new level, the rebuild count is 1) and time
  it (a headless probe).

## Choosing what goes in a preset

Measure each candidate's cost on the High preset with A/B; put the expensive, low-visibility ones in the Low / Medium
cuts first (far SDFGI cascades, far shadow detail, march steps), keep cheap ones that carry the look (tonemapping,
fog, grade) at every level. State the expected fps per preset on the reference machine in the design.
