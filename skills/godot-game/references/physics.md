# Physics (Godot 4, Jolt)

## Engine

- Jolt is built in from Godot 4.4: `physics/3d/physics_engine="Jolt Physics"` in `project.godot`. It is faster and
  more stable for stacks and characters than the default GodotPhysics3D; a few settings differ (check the Jolt
  section of the project settings). The reference project used it on 4.7.2.
- Physics runs at `physics_ticks_per_second` (60 default). Tests use `--fixed-fps 60` so each frame is one tick.

## Characters

- `CharacterBody3D` + `move_and_slide()`; `velocity` in m/s; `floor_max_angle`, `floor_snap_length` for slopes and
  stairs; `is_on_floor()` after the move.
- A movement state machine (WALK, WADING, SWIMMING, ...) chosen from the world each tick (water height at the feet and
  head, floor contact), never from input alone. Ice or a raft as a `StaticBody3D` whose top sits above the water level
  needs no player code: the same state logic sees ground on top and a ceiling below.
- Expose a duck-typed contract (`input_enabled`, `respawn()`, `get_camera()`, `set_look(yaw, pitch)`) for the state
  machine and the UI.

## Collision layers

Write the layer table in the design (name per bit) and use it everywhere, including render `layers` for visuals:

| Bit | Physics use | Render use (example) |
|---|---|---|
| 1 | terrain, rocks, static world | world |
| 2 | water surface trigger | water surface, underwater quad |
| 3 | vegetation (no collision) | vegetation (excluded from particle collision maps) |

## Queries

```gdscript
var q := PhysicsRayQueryParameters3D.create(from, to, mask)
q.collide_with_areas = false
var hit := get_world_3d().direct_space_state.intersect_ray(q)
if not hit.is_empty():
	var p: Vector3 = hit["position"]
```

Run queries in `_physics_process` (the space is locked otherwise). For many cheap height lookups, sample the
heightmap data directly instead of raycasting; a headless world check compares the two at a few points.

## Terrain collision

- `HeightMapShape3D` from the heightmap at full resolution (collision does not need LOD); the visual mesh is chunked
  and LODed separately. Both read the same data file.
- Concave shapes rebuilt at runtime (ice growing) cost a frame spike: rebuild at most every few seconds, in steps, in
  a time-sliced way, and only when the change crosses a threshold.

## AI movement (non-physics)

Many agents (fish, birds, insects) need no physics bodies: steer a position and velocity yourself (desired direction
from the state machine, separation / cohesion / alignment for schools, obstacle avoidance from baked obstacle lists or
a few probes), clamp to the world (water volume, terrain height from data), and integrate in `_physics_process`. Far
agents tick slower (`godot-performance`).
