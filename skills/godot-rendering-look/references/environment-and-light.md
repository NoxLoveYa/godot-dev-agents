# Environment, key light, GI, fog, glow, grade

## Baseline for outdoor realism (Forward+)

| Setting | Value | Why |
|---|---|---|
| `tonemap_mode` | AgX (Godot 4.4+) | filmic highlight roll-off, keeps saturated sunsets from clipping to yellow |
| `tonemap_exposure` | scripted from the look table (`2^EV`) | auto-exposure pumps when the camera turns toward a low sun; keep it off or within ±0.7 EV of the script |
| SDFGI | on for High, `sdfgi_read_sky_light` on, cascades by quality (**2** for most presets) | dynamic GI with a moving sun; see the cascade lesson below |
| SSAO | on Medium+ | contact shadows under vegetation |
| SSR | optional (Ultra); the water does its own reflection march | |
| Volumetric fog | on Medium+; volume size / depth by quality (32 / 64) | god rays through trees from the shadowed key light |
| Glow | on; HDR threshold above the brightest plain surface | bloom only on the sun, its glitter, wet highlights |
| TAA | on (project setting) | stabilises alpha-tested foliage and amortised effects |
| Directional shadow | atlas 2048-4096, soft filter quality by preset, 16-bit | |

## Key light (one DirectionalLight3D)

- One shadowed key light is the sun by day and the moon by night: swap direction while its energy is ~0 in civil
  twilight (sun elevation between -1° and -4°). One light keeps the budget, and the water's sun glitter becomes a moon
  path for free.
- Clamp the light's elevation to ≥ 3° (sun) / ≥ 8° (moon) so shadows do not streak to infinity at the horizon; fade
  energy to 0 over the last few degrees.
- Colour from the sky model's transmittance (the same LUT the sky uses), so sky and light always agree; energy and
  tint from the look table.
- `light_angular_distance` 0.5° clear -> 3-5° overcast (soft shadows); `shadow_opacity` 1.0 day, ~0.85 at golden hour
  (lifted shadows), 0.2-0.35 overcast or rain.
- `visible = false` when energy < ~0.002 (saves the shadow pass on dark nights).
- **Shadow gate (measured):** in a thunderstorm the shadow map was a quarter of the High frame while shadows were
  invisible. Measure "key energy x shadow opacity"; below a cut-on threshold fade opacity to 0 over ~2 s, then set
  `shadow_enabled = false`; above a higher cut-off (hysteresis) bring it back the same way. AND it with the user's
  shadow setting; never gate in moon mode, where weak shadows still read against dark ambient.
- A lightning flash is a separate DirectionalLight3D with shadows off and `light_bake_mode = DISABLED` (no SDFGI
  update), not a child of the node where other code looks for "the sun".

## SDFGI

- `rendering/global_illumination/sdfgi/frames_to_update_lights` (project setting; also
  `RenderingServer.environment_set_sdfgi_frames_to_update_light` live): 4 frames on Medium / High, 8 on Low; a moving
  sun then costs ~0.3 ms.
- **Cascades 4 -> 2 removed most walk spikes** (3-6 ms spikes even standing still); cascades 3-4 only lit the far,
  fogged forest. Keep 4 on Ultra if you like.
- Change `sdfgi_energy` only when the target moves by more than ~0.05; frequent changes re-light.

## Fog

- Depth fog: `fog_sun_scatter` 0.75-0.9 at a low sun, `fog_aerial_perspective` ~0.45, density from the look table x the
  weather multiplier; height fog (`fog_height`, `fog_height_density`) for mist.
- Volumetric fog for god rays: `light_volumetric_fog_energy` on the key light (~2.0 golden hour, 0.8 noon),
  `volumetric_fog_anisotropy` 0.78-0.82 at a low sun (forward scatter: glowing air toward the sun), 0.6 at noon,
  `volumetric_fog_length` 110-160 m (shorter = more froxel resolution near the camera), temporal reprojection on.
- `FogVolume` nodes for local mist over water (needs volumetric fog on).
- Underwater code that toggles `fog_enabled` / `volumetric_fog_enabled` must cache and restore the values the settings
  owner set, and be the only other writer of those flags.

## Glow, grade, streaks

- Glow per look: e.g. golden hour intensity 0.75, strength 0.9, bloom 0.07, HDR threshold 1.05; noon bloom 0.02,
  threshold 1.3.
- No colour LUT blending in the engine (one `adjustment_color_correction` texture, no cross-fade): grade with light
  colours, fog colours, `adjustment_brightness/contrast/saturation`, and a pre-tonemap full-screen quad (white-balance
  gain in a multiply pass, veiling glare and sun streaks in an add pass).
- A depth-occluded sun streak: project the sun, take ~12 depth taps around it; visibility = fraction that hit the far
  plane (reverse-Z: depth 0 = sky in Godot 4.3+); foliage between camera and sun then makes the streak sparkle.

## Full-screen quads in 3D (grade, underwater)

```glsl
shader_type spatial;
render_mode unshaded, depth_test_disabled, skip_vertex_transform;   // (unverified: pick the render modes your pass needs)
void vertex() { POSITION = vec4(VERTEX.xy, 1.0, 1.0); }               // QuadMesh 2x2, reverse-Z near plane (4.3+)
```

Give the MeshInstance3D a huge `extra_cull_margin` so it is never culled, order passes with `render_priority`
(grade 90 before underwater 100), and put it on a render layer you can switch off for A/B. Linear depth from
`hint_depth_texture`: `INV_PROJECTION_MATRIX * vec4(SCREEN_UV * 2.0 - 1.0, depth, 1.0)`, divide by `w`.

## Look table

Keyframes by sun elevation (e.g. -18, -12, -6, -2, 2, 7, 15, 35, 60°), each row: key colour x energy, ambient energy,
EV, fog colour / density, volumetric density / anisotropy, glow bloom / threshold, saturation / contrast, gain, lift /
veil / streak. Morning rows reuse evening rows with a cooler, mistier tint. Weather modifiers multiply (overcast key x
0.22, shadow opacity 0.35, angular 3°, ambient x 1.35, EV +0.3, fog x 1.8 ...). Season tints are small gains. Smooth the
result with a ~0.15 s time constant and expose `snap()`. Test headless: key energy monotonic from -2° to 35°, moon mode at
night, golden-hour key R/B ≥ 1.8, night EV in its band.
