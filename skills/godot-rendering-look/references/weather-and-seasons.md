# Weather, wetness, snow, seasons and ice

## State (owned by one autoload)

- **Weather chain:** a Markov chain over a neighbour graph (CLEAR - PARTLY - OVERCAST - LIGHT RAIN - HEAVY RAIN - STORM,
  MIST from CLEAR / PARTLY in the morning, SNOW from OVERCAST in winter) with Metropolis-Hastings acceptance and visit
  targets `share / mean_dwell`, so long-run time shares equal a per-season table (testable over 200 simulated days).
  Dwell per state in game hours with a floor in real minutes (short days must not strobe the weather).
- **Staged transitions:** worsening ramps clouds first (t 0-0.7), precipitation later (0.45-1), lightning last
  (0.8-1); improving reverses it. Each state defines a parameter row (cloud cover, density, stratus, cirrus, haze, fog,
  mist, rain, snow, wind base and gust, lightning rate); the blend is published.
- **Integrators in real seconds** (surfaces do not speed up with short days): wetness rises with rain and dries with
  sun and wind; puddles lag wetness; snow cover builds below 1 °C; ice forms over minutes in winter. A snap for panel
  changes sets them to the steady state of the new weather and season.
- **Events with one source of truth:** lightning strike times as a Poisson process from the state owner; the FX draws
  the flash and the audio plays thunder after `distance / 343` s from the same signal.
- **Wind coupling:** the weather sets the wind autoload's base speed and gust scale, smoothed (~20 s).

## Globals published

| Global | Meaning | Readers |
|---|---|---|
| `sun_direction`, `moon_direction` (vec3) | unit vectors | sky, foliage backlight, motes |
| `daylight` (float) | 0 night .. 1 day | foliage, plants, ice |
| `season_weights` (vec4) | spring, summer, autumn, winter, sum 1 | terrain, foliage, bark, plants |
| `env_rain`, `env_wetness`, `env_puddles`, `env_snow`, `ice_amount` (float) | 0..1 | water, terrain, rock, foliage, ice |
| `lightning_flash` (float) | 0..4 | sky, sky capture (written by the weather FX module) |
| `water_current` (vec4) | dir.xz, speed, sway | aquatic plants, floating debris |

Shaders skip the wet / snow branches when the global is 0 (uniform branches are cheap).

## Weather FX module

- **Rain:** `GPUParticles3D` following the camera (box ~28 x 1 x 28 m, ~14 m above), fixed `amount` per quality, live
  intensity through `amount_ratio` (4.2+); velocity-aligned streak quads; colour from the look (fog colour), fog-affected;
  reduced under canopy.
- **Collision:** a `GPUParticlesCollisionHeightField3D` following the camera (update when moved), its mask on the
  ground / water layers but not vegetation, so rain falls through canopies; process material collision mode
  hide-on-contact.
- **Splashes (4.7.2):** collision sub-emitters (`SUB_EMITTER_AT_COLLISION`) did not spawn. Kill drops at a height
  texture or the heightfield and spawn splashes from a separate emitter sampling the same heights, or rely on the
  procedural rain rings on water.
- **Drips** under canopy after rain (wetness x forest factor); drip rings on the water under overhanging trees.
- **Snow:** slower flakes with drift, the same follow-the-camera rig.
- **Lightning:** a bolt mesh / shader over the far treeline, a flash envelope written to `lightning_flash`, a separate
  shadowless light; a `reduce_flashing` setting caps the peak and removes repeat pulses (photosensitivity).
- **Mist:** a `FogVolume` over the water, needs volumetric fog; with it off only height fog remains.
- Hide particles when the camera is underwater.

## Seasons on surfaces (shared include `env_surface.gdshaderinc`)

- Terrain: grass colour by season, dry autumn tones, snow on up-facing slopes by `env_snow`, wet darkening and puddles.
- Foliage: broadleaf autumn colours and leaf drop, conifers keep needles and take snow on top faces.
- Rocks and logs: a shader (not `StandardMaterial3D`) so algae below and moss above the water line, wetness and snow
  work; per-instance variation through MultiMesh custom data.

## Ice

- A mesh over the shallows from a mask function `ice_mask(world_xz, depth, amount)` defined once in a shader include
  and **mirrored exactly on the CPU** (same hash) for collision and `is_iced(x, z)`; a test compares the CPU mask with an
  independent port of the shader formula on ~1000 points.
- Collision: a `StaticBody3D` whose top sits above the water level (the player walks on it, swims under it); a 1 m
  collision grid vs a soft visual rim; rebuilt at most every few seconds, at 0.1 steps of `ice_amount`, time-sliced.
- Ripples and disturbances suppressed where iced; swell scaled by `1 - ice`.
