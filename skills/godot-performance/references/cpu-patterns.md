# CPU patterns for GDScript

## Throttling AI

- A "brain" (perception, state machine, desired heading, avoidance) at ~20 Hz per agent, staggered with jitter
  (`_brain_t = BRAIN_INTERVAL * rng.randf_range(0.7, 1.3)`), and the body (integrate velocity toward the desired
  heading, clamp to the world) every physics frame. Urgent states (a strike) think every frame.
- Distance LOD: beyond ~30-45 m from the camera, agents update at 10 Hz and stop their `AnimationPlayer`; never
  throttle an agent that is engaged with the player.
- A manager owns the list and the LOD decision; agents expose `is_throttled()` for tests and stats.
- Neighbour queries for schools from the school's member list (small), not from a global search; obstacles from baked
  lists (rocks, logs, plant beds) in a coarse grid, not raycasts per agent per tick.

## No allocation per frame

- Reuse arrays: `PackedVector3Array` / `PackedFloat32Array` members resized once; `clear()` keeps capacity for typed
  arrays in most cases (measure if it matters).
- No `str()` / `%` formatting, `Dictionary` or `Array` literals, lambdas or `get_nodes_in_group` in hot loops.
- Cache node references and resources in `_ready`; look up optional providers (a module in a group) at most once per
  second, not every frame.
- Write engine properties only on change (Environment setters can trigger work); compare with an epsilon.

## Work at the right rate

| Work | Rate |
|---|---|
| Body motion, camera | every frame / physics tick |
| AI brain | 10-20 Hz, staggered |
| Ambience weights, sun occlusion mirror, floating debris | 4-10 Hz |
| Sky capture, cloud map refresh | amortised per frame, or 4-15 Hz |
| Collision rebuilds (ice) | ≤ every few seconds, time-sliced, on thresholds |
| Statistics, UI text | 4 Hz or on change |

## Many objects

- Hundreds of simple moving things (insects, motes, fry, debris): one `MultiMeshInstance3D` updated from script at a
  low rate, or GPU particles with a process shader; not hundreds of nodes.
- Thousands of static things: baked MultiMesh chunks, never nodes.
- Signals for rare events, globals for continuous values many shaders read.

## Measuring CPU

A headless `-s` probe with `--fixed-fps 60` that runs the system for N frames and times the call with
`Time.get_ticks_usec()` (per call and worst frame), e.g. a 110-fish tick with ice on and off, a collision rebuild, a
quality swap. Report µs per frame against the budget.
