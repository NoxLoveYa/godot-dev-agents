# Architecture patterns (from the reference project)

These patterns let five or six agents build one game in parallel without editing each other's files, and let every
module run alone in a test.

## 1. State owner, look owner, readers

Split each domain into one **state owner** (pure logic, usually an autoload, runs headless) and one **look owner**
(a scene module that turns state into engine fields). Everything else only reads.

```
Climate [-200]  clock, astronomy, weather chain, slow integrators ──► global shader uniforms
   │ signals (time_changed, weather_changed, lightning ...)
   ▼
Atmosphere [-150]  look = table(sun elevation) ⊕ weather ⊕ season ──► sun light, Environment, sky material
Weather FX [-140]  particles, lightning flash ──► lightning_flash global
Water, life, fish, ambience [0]  read state and globals
Settings [1000]  enforces the quality fields last
```

Numbers in brackets are `process_priority`. The state owner has one step function, `advance(real_seconds)`, that
`_process` calls and tests call directly with large steps.

## 2. Global shader uniforms as the contract

- Declare every cross-module shader value in `project.godot [shader_globals]` with a **default equal to the default
  start state**, so every shader compiles and looks right before its writer exists:

```ini
[shader_globals]
sun_direction={ "type": "vec3", "value": Vector3(-0.34, 0.13, -0.93) }
env_wetness={ "type": "float", "value": 0.0 }
season_weights={ "type": "vec4", "value": Vector4(0, 1, 0, 0) }
```

- Shaders declare `global uniform vec3 sun_direction;` (a shared `.gdshaderinc` with only the declarations keeps them
  in one place). Writers call `RenderingServer.global_shader_parameter_set("env_wetness", v)`.
- The design doc holds a table: name, type, range, default, **writer**, readers. One writer per global.
- Per-material values (a sky's cloud offset, a water's sky texture) stay material uniforms, set by their owner.
- Per-instance values: `instance uniform` works for single MeshInstance3D nodes; MultiMesh instances use
  `INSTANCE_CUSTOM` (custom data) instead.

## 3. Ownership table of engine fields

Write who sets each field of shared engine objects (Environment, the sun DirectionalLight3D, CameraAttributes, bus
effects, a water node's exports). Example rows:

| Field | Owner | Notes |
|---|---|---|
| `Environment.fog_enabled`, `volumetric_fog_enabled` | Water (underwater toggle) and Settings only | look code never writes enable flags |
| `fog_density`, `fog_light_color`, `volumetric_fog_*` amounts | Atmosphere | |
| `sdfgi_enabled`, `ssao_enabled`, shadow mode | Settings | |
| `Sun.rotation`, `light_color`, `light_energy`, `shadow_opacity` | Atmosphere | `shadow_enabled` stays Settings' (gate ANDs with it) |
| Bus volumes | Settings | effects owned per bus by one module |

Two writers of one field show up as flicker or settings that "do not stick". Finding one is an audit must-fix.

## 4. Optional modules as scenes (EnvModules loader)

A small node in the main scene instantiates each optional module scene if its file exists, deferred, in a fixed order:

```gdscript
class_name EnvModules
extends Node
const MODULES := ["res://scenes/env/atmosphere.tscn", "res://scenes/env/weather.tscn", "res://scenes/life/pond_life.tscn"]

func _ready() -> void:
	for p: String in MODULES:
		if ResourceLoader.exists(p):
			get_parent().add_child.call_deferred((load(p) as PackedScene).instantiate())
```

- Parallel packages each ship one scene; nobody edits the main scene during the wave.
- Tests can skip modules (a `skip` list set before the node enters the tree).
- Rules for modules: extra lights are not direct children of Main when other code takes "the first
  DirectionalLight3D child" as the sun; nodes that Settings scales by name must be where Settings looks for them.
- A module that must rebuild on a quality change can be re-instanced by the loader (hand over expensive caches).

## 5. Duck-typed adapters

A gameplay system (fish AI) talks to the world through one adapter (`FishWorld`) that looks things up by group and
`has_method`, with fallbacks. The same system then runs in a tiny test scene with a mock player and a flat terrain,
and in the real game, unchanged. New world features (cover, ice, time of day) are added to the adapter as optional
queries that return a neutral value when the provider is missing.

## 6. GameFlow state machine (autoload)

`BOOT -> MENU <-> PLAYING <-> PAUSED`. It alone owns: `get_tree().paused`, `Input.mouse_mode`, enabling and hiding the
player (`process_mode = DISABLED` in menus), respawn, fades (through the UI root that registers itself), and audio
ducking calls. It emits `state_changed(new, old)`. `start_game()`, `pause()`, `resume()`, `to_main_menu()` are guarded
by a `busy` flag during fades. Systems that only advance during play check `GameFlow.is_playing()` (tests must start
the game before they expect time to run).

## 7. Settings autoload with presets

- `DEFS`: key -> `{sec, def, min, max | values}`; one table drives load, save, clamping and the options UI rows.
- `PRESETS`: preset -> values for `PRESET_KEYS`; changing one of those keys by hand switches the preset to CUSTOM.
- `changed(key, value)` signal; `reloaded` after a load.
- Applies world-dependent settings lazily and again whenever a WorldEnvironment, a vegetation builder or the player
  enters the tree (`get_tree().node_added`), and runs at `process_priority` 1000 when it must win over a module that
  rewrites a field every frame.
- `cfg_path` is a var (tests point it at a temp file). Details: `saving-and-settings.md`.

## 8. Look and tuning tables

Values that vary with a continuous input (sun elevation, depth, distance) live in a keyframe table with linear
interpolation (`LookTable.sample(elevation)`), modifiers per discrete state (weather) multiply on top, and the result
is smoothed (time constant ~0.15 s) with a `snap()` for tests and shots.

## 9. Data generators outside the game

Terrain, scatter and placement come from deterministic Python generators that write `.bin` / `.json` into
`assets/` and the constants a script needs (`world_config.gd`, a marked block they rewrite). The game only reads.
A second generator for a new layer is safer than adding draws to the first generator's RNG sequence (any draw added
before the very end shifts every later draw, so existing layers move).
