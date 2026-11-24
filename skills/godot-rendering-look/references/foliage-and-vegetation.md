# Foliage shaders and vegetation at scale

## Foliage shader

```glsl
shader_type spatial;
render_mode cull_disabled, diffuse_burley, specular_schlick_ggx;
global uniform vec2 wind_direction;
global uniform float wind_strength;
global uniform float wind_speed;
global uniform vec3 player_position;
global uniform vec4 season_weights;
uniform sampler2D albedo_tex : source_color, filter_linear_mipmap_anisotropic;
uniform float alpha_cut = 0.45;
```

- **Wind** in `vertex()`: world-space displacement from the global wind (one writer: a Wind autoload with smooth noise
  for swell, gusts and rare strong spells), converted back to local space. Vertex colour channels as masks baked by the
  asset generator: r = flex, g = per-element phase, b = height, a = tip. Trees: trunk sway, branch sway, leaf flutter;
  grass: bend; push-away around `player_position`.
- **Alpha:** `ALPHA_SCISSOR_THRESHOLD = alpha_cut` (opaque pipeline: depth prepass, shadows, works behind water
  refraction). Alpha hash (`ALPHA_HASH_SCALE`) for soft fades; alpha-to-coverage (`render_mode alpha_to_coverage` with
  `ALPHA_ANTIALIASING_EDGE`) needs MSAA. TAA hides most scissor shimmer.
- **Translucency:** the engine's `BACKLIGHT` output lights cards from behind with the light's colour and shadows; boost
  it toward a low sun (forward-scatter golden-hour rim) with a gain uniform (0 = old look, for A/B). Tree crowns, many
  overlapping cards already lit from both sides, take only a fraction of the boost.
- **Variation:** per-instance tint from a hash of the instance position or `INSTANCE_CUSTOM`; autumn colours and leaf
  drop from `season_weights` (drop a whole card at once by a per-card hash, not per pixel: per-pixel dissolve shimmers
  under TAA).
- Conifers vs broadleaf (a `deciduous` uniform) so winter strips only broadleaf; reeds keep standing while meadow grass
  flattens under snow.

## Vegetation at scale

- **Baked scatter** (a deterministic generator writes transforms per layer into `.bin`), **chunked
  `MultiMeshInstance3D`s** (e.g. 40 m cells) so frustum culling works per chunk; a coarse far layer (the same trees in
  100 m cells, only the cheapest LOD beyond ~200 m) keeps a 400 m forest at few draw calls.
- **LOD by visibility range**, one MultiMesh per LOD per chunk: `visibility_range_begin`, `visibility_range_end`,
  `visibility_range_begin_margin`, `visibility_range_end_margin`, `visibility_range_fade_mode`.
- **Fade margins draw two LODs at once (measured):** with `VISIBILITY_RANGE_FADE_SELF`, both LODs render (dithered)
  across the margin. With 7 / 16 / 24 m margins that was ~41k triangles of overlapping alpha-tested cards per tree in
  the 16-47 m ring: **5-9 ms of the High frame in the forest**. Keep margins short on dense layers (scale them by
  preset: e.g. 0.3 of the designed margin on High) or disable fade there.
- **Shadow proxies:** visible LODs do not cast (`cast_shadow = OFF`); separate shadow-only instances
  (`SHADOW_CASTING_SETTING_SHADOWS_ONLY`) cast with a cheaper LOD per distance band, and beyond ~55 m a closed **crown
  hull** (a convex blob from the LOD's bounds, a trivial shader) replaces leaf cards: the far alpha-tested cards cost
  3.5-4.5 ms of shadow-pass fill at a low sun; hulls cost a fraction, and 55 m-away shadows are crown blotches at that
  shadow resolution anyway.
- **Imported meshes are shared:** builders set materials on private `duplicate()`s of the meshes, or a second builder
  of the same file replaces the first builder's materials.
- **Settings scaling:** density (instance counts by layer name) and view distance (visibility ranges x factor), applied
  by Settings to builder nodes it finds by a known place in the tree and layer names in a list. New layers register
  their names there.
- Short visibility ranges for dense ground cover (grass 50 m, flowers and ferns 55 m), larger for shrubs and rocks;
  rocks cast real shadows, grass none.
- **Aquatic plants:** clumps instead of single blades, alpha scissor, no shadows, a 40 m range for dense ribbon beds,
  a travelling wave along the blade driven by a `water_current` global (direction, speed, sway), wave only near the
  camera on lower quality.

## Terrain

- Heightmap from the generator (`.bin`, e.g. 801² at 0.5 m), visual chunks with LOD (0.5 / 1 / 2 / 4 m by distance)
  and skirts against cracks; collision at full resolution from the same data.
- Splat by slope, height and masks in the terrain shader; season, wetness (darker, glossier, puddles in flat low spots)
  and snow (on up-facing slopes) from the globals.
