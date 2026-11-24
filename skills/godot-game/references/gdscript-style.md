# GDScript style and typing (Godot 4.x)

## Typing

- Static types everywhere: `var speed: float = 3.0`, inferred `var d := Vector3.ZERO`, typed returns
  (`func f() -> void`), typed arrays (`Array[StringName]`), typed loops (`for p: String in paths:`). Typed code is
  faster and catches errors at parse time.
- Typed dictionaries `Dictionary[String, int]` exist from 4.4 on; older code uses `Dictionary`.
- `StringName` literals (`&"fov"`) for keys, actions, groups and signals; `NodePath` literals (`^"Path"`) rarely needed.
- Cast on the way out of untyped APIs: `(load(p) as PackedScene).instantiate()`, `node as Camera3D`.
- Enable the warnings for untyped declarations in the project settings (`debug/gdscript/warnings/untyped_declaration`)
  when the team wants it enforced.

## Shape of a script

```gdscript
class_name PondFish
extends CharacterBody3D
## One fish: steering brain at 20 Hz, body motion every physics frame.
## Contract: lives under the node in group "fish_manager"; reads the world through FishWorld (duck typed).

signal state_changed(new_state: int)

enum State { WANDER, FLEE, HIDE }

const BRAIN_INTERVAL := 0.05          ## s between brain ticks (jittered per fish)

@export var species: FishSpecies
@export_range(0.0, 5.0, 0.1) var cruise_speed := 0.6

var state: State = State.WANDER
var _brain_t := 0.0

@onready var _anim: AnimationPlayer = %AnimationPlayer
```

Order: `class_name`, `extends`, `##` doc comment, signals, enums, constants, exports, public vars, private vars
(`_name`), `@onready`, built-in callbacks, public methods, private methods. Doc comments with `##` show in the editor.

## Per-frame code

- No allocation in `_process` / `_physics_process` on hot paths: no new Arrays / Dictionaries / Strings per frame, no
  `str()` formatting, no `get_nodes_in_group` per frame. Cache node references; reuse `PackedFloat32Array` buffers.
- Heavy thinking at a lower rate: an accumulator per object, staggered with jitter so N objects do not tick in the
  same frame (`_brain_t = BRAIN_INTERVAL * rng.randf_range(0.7, 1.3)`).
- Distance LOD for behaviour: far objects update at 10 Hz, stop animating, skip avoidance.
- Write engine properties only when they change by more than an epsilon (Environment and light setters are not free;
  some trigger GI or shadow updates).
- `RenderingServer.global_shader_parameter_set` is cheap per frame; still skip it when the value did not change.

## Randomness and determinism

- One `RandomNumberGenerator` per system, seeded (`rng.seed = 1234`), its `state` saved when the system persists.
  `randf()` globals are shared and unseeded: avoid them in anything a test checks.
- Tests set seeds **before** deferred spawns (a manager that spawns in `call_deferred` reads its seed then).

## Async

- `await get_tree().process_frame`, `await get_tree().create_timer(t).timeout`, `await tween.finished`. A function
  with `await` is a coroutine: callers that need the result `await` it too.
- In a `SceneTree` test script, `await process_frame` (the script is the tree).
- `Tween`: `create_tween()` per animation, `kill()` the previous one before starting another on the same property.

## Errors

- `push_error` / `push_warning` for problems (they appear as `ERROR:` / `WARNING:` lines in headless logs and fail the
  regression runner's log scan); `assert()` only for programmer errors (stripped in release).
- Never swallow an error to make a test pass; never disable a feature to silence a warning.

## C#

The same architecture applies; C# needs the .NET build of Godot and `dotnet build` before headless runs. Keep tool
and test scripts in GDScript even in a C# project: `-s` scripts and quick probes are simpler there.
