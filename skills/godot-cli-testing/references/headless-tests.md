# Headless tests

## Kinds

| Kind | Setup | Example checks |
|---|---|---|
| **Pure logic** | throw-away instances (`AppClimate.new()`, not in the tree), seeded, own save path | astronomy values at named times, weather time shares over 200 simulated days within ±5 points, integrators reach their targets, save -> change -> load round-trip, corrupt file -> defaults |
| **Module in a test scene** | a small `.tscn` (flat terrain, mock player) plus the module | AI bounds, flee / hide durations, no agent above the water |
| **Main scene integration** | `main.tscn` instantiated, game started through the state machine | node layout and order, modules loaded, menus reachable, Settings changes reach the nodes |
| **Contract** | load an asset or scene, inspect it | skeleton bone count, animation names / lengths / loop modes, AABB, node names other code looks up |
| **Data** | read generator outputs | counts within ±15% of the design, every instance inside its band, determinism (two runs, same bytes) |

## Rules that came from failures

- **Determinism:** `--fixed-fps 60` for anything with time; seed RNGs **before** deferred spawns (set a
  `deterministic = true` flag on spawners from `_initialize`, before the scene enters the tree); compare against
  simulated time (`Engine.get_physics_frames() / physics_ticks_per_second`), never `Time.get_ticks_msec()`.
- **State machines:** a headless main-scene test must start the game (`GameFlow.start_game()` or the state entry it
  uses) before it expects clocks or AI to run.
- **Autoloads:** `root.get_node("Climate")`; the test sets `persist = false`, a temp `save_path`, and disables
  processes that would add noise (`wind.set_process(false)`) before it loads anything.
- **No real files:** never write the player's `user://` files, never write `res://`; delete temp files at the end.
- **Statistics:** test long-run distributions with many simulated days through the pure `advance(dt)` step, not by
  running frames.
- **Thresholds from the design:** every check's numbers come from the design doc's acceptance list; print the measured
  value next to PASS / FAIL (`"%.2f" % v`), so a near miss is visible.
- **Ports of shader maths:** when CPU code mirrors a shader (an ice mask, a swell height), the test compares the CPU
  function with an independent GDScript port of the shader formula on ~1000 points.
- **Physics checks:** place a body, step N physics frames, check its state (a player on ice over deep water stays in
  WALK with `y >= -0.05` for 120 frames).
- **Flaky checks:** fix the cause (usually wall time or an unseeded RNG). A runner `-Retry` is a stopgap for one known
  flaky check, named in its README.

## Driving UI headless

- Open pages through the UI root's API or by injecting input:
  `var ev := InputEventAction.new(); ev.action = &"ui_accept"; ev.pressed = true; Input.parse_input_event(ev)`,
  then a frame later the release event.
- Check focus with `root.gui_get_focus_owner()` (or `get_viewport()` from a node), visibility, and the values the page
  wrote (Settings, the state owner) while the tree is paused.

## Assertion helper with context

```gdscript
func near(name: String, v: float, lo: float, hi: float) -> void:
	check(name, v >= lo and v <= hi, "%.3f in [%.3f, %.3f]" % [v, lo, hi])
```

## NaN scan

A helper that walks floats, vectors, colours, bases, transforms, arrays and dictionaries and reports any `is_nan` /
`is_inf`, applied to the state owner's public state and the values it publishes as globals. Cheap; use it in soaks and
integration tests.
