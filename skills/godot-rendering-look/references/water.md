# Water: surface, refraction, ripples, caustics, underwater

Built at runtime by one water node (surface mesh sized from world config, ripple simulation, caustics, underwater
volume), configured by `@export`s that a look director may scale (base x factor, bases read once on its first frame).

## Surface shader

```glsl
shader_type spatial;
render_mode unshaded, cull_disabled, depth_draw_never, blend_mix;
uniform sampler2D screen_tex : hint_screen_texture, repeat_disable, filter_linear_mipmap;
uniform sampler2D depth_tex : hint_depth_texture, repeat_disable, filter_nearest;
uniform sampler2D sky_tex : repeat_enable, filter_linear;      // equirect sky capture
uniform sampler2D sim_tex : repeat_disable, filter_linear;     // ripple height field (camera-centred window)
uniform sampler2D depth_map : repeat_disable, filter_linear;   // baked water depth over the pond
```

- **Unshaded, own lighting**: analytic sun glitter (a wide lobe broken into sparkles by sub-decimetre facets scaled by
  wind), sky reflection from the capture (blurred by roughness with a few taps), a screen-space reflection march for
  the banks (steps by quality), Fresnel.
- **Refraction**: offset `SCREEN_UV` by the surface normal, read `screen_tex`; check the refracted depth so objects in
  front of the water do not bleed in.
- **Absorption by path length**: linear depth of the bottom under the refracted ray minus the surface depth gives the
  path; transmission `exp(-sigma * path)` per channel, plus in-scatter toward `scatter_color`. A **depth-based pair**
  blends a warm shallow colour into the deep colour by the water depth at the seen bottom point
  (`s = 1 - smoothstep(0.6, 2.8, d_bottom)`; mix sigma and scatter), so shallows read green-gold and deep water teal.
  Sample the depth at the bottom point, not the surface point, or the shelf edge shows a band.
- **Swell** in the vertex shader: a few long waves (wavelength, amplitude weight, direction offset, phase) mirrored by a
  CPU function (`get_water_height`) for floating objects and the player; published as a global (`water_swell`) so
  lilies and debris bob in phase.
- Procedural rain rings from hashed cells (several layers, ring speed ~0.35 m/s, life ~0.9 s) scaled by the `env_rain`
  global, faded beyond ~30 m; the same function gives puddle ripples on terrain.

**Alpha-blended objects behind the water vanish** **(4.7.2)**: the screen texture is captured after the opaque pass, so
transparent (blended) materials are not in it. Use alpha scissor or alpha hash for anything that must be seen through
the water (plants, particles, fish fins).

## Ripple simulation (interactive)

- A wave equation on a height texture in a camera-centred window (e.g. 48 m at 0.075 m texels), run in `SubViewport`s
  with a `canvas_item` shader that reads the previous two states (ping-pong), set to `UPDATE_ONCE` each frame. The
  window recentres in whole texels.
- **Stability:** the explicit scheme on 0.075 m texels was stable up to ~1/23 s per step; a longer frame is split into
  equal sub-steps (at most 3; below ~8 fps ripples slow down). Sub-step chains are separate viewports.
- **(4.7.2, probed)** SubViewports set to `UPDATE_ONCE` in the same frame rendered in **creation order**; build each
  chain front to back, and keep a probe test if you rely on it.
- Disturbances (`add_disturbance(pos, strength, radius)`) are queued and drawn into the next step; ignore them where
  ice covers the water.
- The simulation pauses with the game; lighting of the water (sun, colours, caustics) keeps updating while paused
  (`PROCESS_MODE_ALWAYS` on the node, the sim gated on `get_tree().paused`). A windowed test checks ripple speed at 60,
  30, 20 and 10 fps.

## Caustics

Procedural (animated Voronoi-like or a baked caustic texture) projected from the sun direction onto everything below
the water level, in the receiving materials or the underwater pass, with chromatic split by quality (1 or 3 channels),
scaled by key light energy (none on overcast).

## Underwater

- When the camera goes below the surface (camera height vs `get_water_height`), show a full-screen quad pair: a
  multiply pass (absorption along the view ray, the same sigma as the surface) and an add pass (in-scatter, sun shafts
  marched through the shadow map or analytic, steps by quality), hidden above water.
- **Snell window:** looking up, the sky is visible only inside ~48.6° of the vertical; outside it the surface mirrors
  the underwater scene (total internal reflection), which a short underside SSR march or a flat dark colour can give.
- Toggle fog / volumetric fog for underwater only as an owner of those flags (cache and restore), switch a low-pass on
  the ambience bus, enable depth of field if the look needs it.
- Silt: a small GPUParticles3D around the camera underwater.
