# Code review checklist (Godot)

Read the connected code, not only the diff. Rank findings must fix / should fix / nice to have, each with
`path:line`, what goes wrong and when, the fix.

## Correctness
- [ ] Node paths, groups, signal names and method names the change uses exist, and every duck-typed caller of a
      renamed one is updated (grep the whole project, including `tools/`).
- [ ] Signals connected once; long-lived emitters disconnected from freed listeners; no work on freed nodes
      (`is_instance_valid`).
- [ ] `_ready` order, `call_deferred` where a parent is busy, `await` chains that can resume after the node is gone.
- [ ] Process mode correct for the pause; `process_priority` matches the data flow in the design.
- [ ] Save / load validates and clamps; versioned; temp-file write then rename for structured saves.

## Contracts and ownership
- [ ] The change writes only fields and globals its module owns (design ownership table).
- [ ] New global shader uniforms are declared in `[shader_globals]` with defaults; shaders compile without the writer.
- [ ] New data files have a reader contract (format, header, version) and are in the export include filter.

## Performance
- [ ] No allocation, string formatting or group lookups per frame on hot paths.
- [ ] Throttled AI / thinking with jitter; distance LOD for behaviour and animation.
- [ ] Engine setters only on change; no per-frame sky radiance updates; no new shadow casters on dense layers.
- [ ] Visibility ranges and fade margins sized for the layer's density.

## Tests
- [ ] Risky logic has a headless test that can fail (an assertion that would catch the bug it guards).
- [ ] Tests seed RNGs, use `--fixed-fps` when they simulate time, wait in frames not seconds, write no `user://` or
      `res://` files.
- [ ] The regression runner lists new tests.

## Export
- [ ] No `DirAccess` listings of resource folders (use `ResourceLoader.list_directory`).
- [ ] Dev scenes and tools excluded; credits and licences for new third-party files.

## Cuts
- [ ] Dead code, unused resources, duplicate helpers, abstractions with one user.
