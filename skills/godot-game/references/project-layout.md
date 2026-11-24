# Project layout, scenes, nodes, signals, autoloads, resources

## Folders

Keep the Godot project in a subfolder of the repo, and dev tooling outside it, so tooling never ships and an export
filter mistake cannot pull it in:

```
<repo>/
├─ <project>/                 the Godot project (project.godot lives here)
│  ├─ scenes/                 .tscn by feature: main.tscn, ui/, env/, life/, player/ ...
│  ├─ scripts/                .gd by feature; scripts/autoload/ for singletons
│  ├─ shaders/                .gdshader, .gdshaderinc by feature (sky/, post/, env/ includes)
│  ├─ assets/                 generated and sourced data: models, textures, audio, data files (.bin, .json)
│  ├─ CREDITS.md              shipped credits (read at runtime by a credits screen)
│  └─ export_presets.cfg
├─ tools/<area>/              headless tests, shot harnesses, generators (Python), PowerShell runners, README.md each
│  └─ out/                    tool outputs (gitignored)
├─ docs/design/               design docs; docs/credits_pending/ during parallel waves
└─ AGENTS.md
```

- `tools/<area>/README.md` lists every tool with its exact command line. Agents read it before running anything.
- Dev and test scenes that must live inside `res://` (a fish test pond, a UI test scene) are excluded in the export
  preset's `exclude_filter`.

## Scenes and nodes

- A scene is a reusable tree saved as `.tscn`; its root's script is the scene's API. Instance scenes; do not build
  large trees in code unless the content is generated from data (vegetation from scatter files, ripples, UI rows).
- Look up children with `%UniqueName` (scene-unique names) or `@onready var x := $Path`, never long `get_node` strings
  across scene boundaries. Across modules use **groups** (`add_to_group("water")`,
  `get_tree().get_first_node_in_group("water")`) and duck typing.
- `_ready` runs children first. Adding a node from a parent's `_ready` while the parent is still being set up fails
  ("parent busy"): use `add_child.call_deferred(node)`.
- `queue_free()` frees at the end of the frame; `free()` immediately (only for nodes outside the tree, e.g. test
  instances). Check `is_instance_valid()` before touching something that may have been freed.

## Process order and pause

- `process_priority`: lower runs first, per frame. Use it for data flow: the state owner (e.g. a Climate autoload at
  -200) before the look director (-150) before readers (0) before an enforcer that must win (Settings at 1000).
- `process_mode`: `PROCESS_MODE_ALWAYS` for menus, the state machine, and anything that must update while
  `get_tree().paused = true` (a pause-menu preview of time or weather); `INHERIT` (default) freezes with the pause;
  `DISABLED` parks a node (a hidden player in the main menu).
- `_physics_process` runs at `physics/common/physics_ticks_per_second` (default 60); with `--fixed-fps 60` frames and
  physics ticks match one to one, which makes tests deterministic.

## Signals

- Declare typed signals (`signal weather_changed(from: int, to: int, duration_s: float)`); connect with callables
  (`x.changed.connect(_on_changed)`). Connect once (check `is_connected` if code can run twice) and disconnect when a
  long-lived emitter outlives the listener.
- Rate-limit signals that fire per frame (emit at most N Hz, only when the value moved by more than an epsilon).
- Prefer a signal from the owner over polling when the change is rare; poll (at a few Hz) when many readers need a
  continuously changing value, or publish it as a global shader uniform.

## Autoloads

- Declared in `project.godot [autoload]` (`Name="*res://scripts/autoload/name.gd"`; `*` = instantiate as a node). They
  load in the listed order, before the main scene; later autoloads may use earlier ones in `_ready`.
- Give each autoload a `class_name` different from its node name (`class_name AppSettings`, node `Settings`) and a
  static accessor, so code and test scripts reach it without the global identifier:

```gdscript
class_name AppSettings
extends Node

static var _inst: AppSettings

static func get_instance() -> AppSettings:
	return _inst if is_instance_valid(_inst) else null

func _enter_tree() -> void:
	if is_instance_valid(_inst) and _inst != self:
		queue_free()   # a second copy (a test scene's own node): keep the first
		return
	_inst = self
```

- Autoloads hold state and rules, not scene content. Pure logic autoloads (clock, weather chain, settings) run
  headless and expose one step function (`advance(seconds)`) that tests call directly.

## Resources

- `Resource` subclasses with `@export` fields are data (species, presets, item definitions): `.tres` files edited in
  the inspector or generated. Resources are shared by every user that loads the same path: `duplicate()` before you
  change one per instance (meshes from an imported `.glb` included).
- Large generated data (heightmaps, scatter transforms, noise volumes) is usually better as raw `.bin` (header + packed
  floats) read with `FileAccess.get_buffer` / `PackedByteArray.to_float32_array()`: no import step, fast, diffable
  size. Such files must be in the export `include_filter` (`workflows/export-checklist.md`).
- `ResourceLoader.exists(path)` tests for a resource without loading it; `load()` caches; `preload()` resolves at
  compile time (only constant paths).

## Groups worth standardising

`player`, `water`, `terrain`, and one per optional module (`atmosphere`, `weather`, `ice`, `pond_life`). Write the
group names into the contract section of the design doc.
