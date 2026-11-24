# Sky shader, atmosphere and volumetric clouds

## Options

| Option | Realism | Cost (1080p) | Rules out |
|---|---|---|---|
| `PhysicalSkyMaterial` | good gradients, flat sunsets | ~0.1 ms | clouds, moon, weather blending |
| HDRI panoramas blended by time / weather | photographic at the captured moments | low GPU, 64 MB per 4k map | a moving sun (baked sun positions), clean transitions, animated clouds |
| **Custom sky shader on a baked LUT + procedural clouds** | physically based colour, all weathers, moon and stars | ~0.15-0.35 ms sky itself (measured, see below) | photographic cloud detail (mitigated by good noise and lighting) |

## Atmosphere LUT (baked offline)

- A Python (numpy) baker, ~1-2 min: planet radius 6360 km, top 6460 km; Rayleigh β = (5.802, 13.558, 33.1)·10⁻⁶ m⁻¹,
  H = 8 km; Mie scattering ~4·10⁻⁶ x haze, absorption 4.4·10⁻⁶, H = 1.2 km, g = 0.8; **ozone** absorption
  (0.650, 1.881, 0.085)·10⁻⁶ m⁻¹ in a tent around 25 km (ozone keeps the twilight zenith blue instead of grey);
  single scattering plus Hillaire's isotropic multiple-scattering term.
- Output a raw `.bin` (header + float16): sky radiance per unit sun for view elevation (64, dense near the horizon) x
  sun elevation (32, -20°..90°) x azimuth difference (32, 0..180°) for two haze levels, plus a transmittance table for
  the sun colour. Load it at runtime into `ImageTexture3D`s (no import pipeline) and read it on the CPU for the key light.
  The `.bin` must be in the export `include_filter`.

## Sky shader structure

```glsl
shader_type sky;
uniform sampler3D lut_clear; uniform sampler3D lut_hazy;
uniform sampler2D cloud_map : filter_linear, repeat_disable;   // the amortised octahedral cloud map
global uniform vec3 sun_direction;
void sky() {
	vec3 d = EYEDIR;
	vec3 col = atmosphere(d, sun_direction);                   // LUT lookups
	vec4 cl = sample_clouds(d);                               // octahedral lookup, no marching here
	col = mix(col, cl.rgb, cl.a);
	if (!AT_CUBEMAP_PASS) { col += sun_disc(d) + moon(d) + stars(d); }
	COLOR = col;
}
```

- **No sun disc, stars or moon in the cubemap pass** (`AT_CUBEMAP_PASS`): radiance must not hold the sun, or
  reflections get fireflies and double highlights.
- Sun disc 0.53° with limb darkening, HDR (drives bloom), dimmed by cloud transmittance; moon ~0.9° (slightly enlarged
  for readability) with phase lighting from the sun direction; stars from hashed cells, rotated by a basis uniform,
  faded by daylight and cloud.
- Below the horizon: a dark ground colour blended to the fog colour.

## Radiance: the TIME trap

- A sky shader that uses `TIME` makes the engine re-render the radiance cubemap every frame **(4.7.2)**. Do not
  animate the sky shader with `TIME`; animate the inputs from script (cloud offset uniform) and amortise.
- `Sky.process_mode = PROCESS_MODE_INCREMENTAL` (radiance updated over several frames; ambient and reflections change
  slowly anyway). REALTIME cost ~0.35 ms here; INCREMENTAL ~0.05 ms. `radiance_size` 256 on Medium+, 128 on Low (with
  INCREMENTAL the precise filter's cost is the radiance size).
- `rendering/reflections/sky_reflections/fast_filter_high_quality` only affects REALTIME skies; it does nothing for an
  INCREMENTAL sky and is read once at startup.

## Volumetric clouds, amortised (Nubis-style)

- **March** a spherical shell (planet radius 6360 km, shell ~1.5-5.5 km above the observer) in a separate
  `canvas_item` shader drawn into an **octahedral map of the upper hemisphere** in a `SubViewport` (HDR, never cleared:
  1024² High, 640² Medium, 320² Low).
- **Amortise:** each frame only 1 texel of every 4x4 block is marched (Bayer order so consecutive frames touch
  far-apart texels; the others `discard` and keep their value), so the map refreshes every 16 frames and the march
  costs a fraction of a millisecond. The sky main pass, the radiance cubemap and the water's sky capture only
  **sample** the map.
- **Temporal accumulation:** two viewports ping-pong (one reads the other as history), a fresh texel blends into its
  history with weight ~0.3; jitter per texel (interleaved gradient noise) and per update (golden-ratio sequence).
- **Noise baked offline** (numpy, tileable, `.bin` volumes): base 128³ (R Perlin-Worley, GBA Worley fbm), detail 32³
  Worley, a 128² curl texture, a weather map 96x96x32 (coverage, type, storm; **z = time**, so clouds form and dissolve
  over minutes).
- **Shape:** `base = remap(perlin_worley, -(1 - worley_fbm), 1, 0, 1)` x the height profile of the cloud type (stratus
  to cumulonimbus), `coverage = remap(base, 1 - coverage, 1, 0, 1)`, detail erosion only at the edges (distorted by
  curl), inverted toward the base for wispy bottoms. Step length grows with distance (`u^1.4`; `u^1.6` aliased into fur
  at thin far edges).
- **Lighting:** Beer-Lambert with multiple-scattering octaves (a = b = c = 0.5), dual-lobe Henyey-Greenstein
  (g 0.8 / -0.3), a powder term, a 6-sample light cone plus one long sample, ambient from the LUT, distant clouds fading
  into the LUT sky (aerial perspective), the Earth's shadow after sunset.
- **Closed decks (overcast, rain, storm):** do not raise coverage on the cumulus noise: thresholds always leave thin
  spots that show bright sky (white streaks, glowing rims, "specular" holes). Blend in a **slab** whose base / top follow
  a smooth thickness field: opaque everywhere, near-isotropic phase (no silver rims), no powder, much lower extinction on
  light rays, darker where thicker, a ragged base and scud under rain decks.
- **Night:** fade the lit radiance of far samples and let aerial perspective give unlit far clouds the sky's colour,
  so the horizon bank is a dark silhouette, not a bright scalloped band.
- **CPU mirror** for gameplay and light: march the sun ray through the same weather map at ~10 Hz on the CPU (no detail)
  to get `sun_occlusion`, which dims the key light when a cloud crosses the sun.
- **Measured** (1080p, interleaved ON / OFF in one process): bare sky +0.14 / +0.18 / +0.35 ms (Low / Medium / High,
  partly cloudy); in the full scene the sky cost +0.5-0.6 ms, mostly ambient / reflection sampling and SDFGI sky light,
  not the clouds.

## Sky capture for custom water

A `SubViewport` (`use_hdr_2d`, 512x256; 256x128 on Low) with a `ColorRect` and a `canvas_item` shader that evaluates the
same sky include per equirect texel (no sun disc: the water draws its own glitter), updated with
`UPDATE_ONCE` at 4-15 Hz by quality. The water samples its `ViewportTexture` with the same equirect mapping
(`u = atan(d.x, -d.z) / TAU + 0.5`, `v = acos(d.y) / PI`).
