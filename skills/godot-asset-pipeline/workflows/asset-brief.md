# Asset brief template (gd-model-lead -> gd-modeller)

Buildable with no questions, checkable from numbers and renders. One asset (or one clip) per brief.

```
# Asset: <id> (<one-line idea>)

Route: <generator script | Blender script via MCP | library asset + edits>, because <why>.
Goal: <what it is in the game, where it is seen, from how far>.

Frame
- Units metres, +Y up (glTF), facing <-Z>, origin <on the ground under the body | at the base centre>.
- Bind pose: <pose>.

Shape (numbers)
- Overall: length <m>, height <m>, width <m>.
- Profile / sections: <table of stations: position, half-width, half-height> or <SDF parts: name, centre, radii, blend>.
- Details that must read at <distance>: <list>.
- Never: <the faults to avoid: stacked cylinders, mirrored seams, floating parts, ...>.

Budget
- Triangles: <n> (LOD0); LODs: <none | n levels with ratios>.
- Textures: <2048> albedo / normal / ORM, <VRAM compressed>; small exact textures: <list>.

Materials
- <slot>: <albedo idea, roughness, double-sided?, alpha scissor?>.

Rig and animation (if any)
- Bones (names exactly): <list>.
- Animations: <name> <length s> <loop | once>, <what moves>.

Files
- Writes: <assets/.../id.glb>, <tools/.../generate_id.py>, <patch_import.py>.
- Reads only: <shared kit / common writer>. Never edits: <shared files>.

Import
- `.glb.import`: loop modes <...>, `meshes/generate_lods=<false>`; extracted textures `compress/mode=2`, normal map
  flagged.

Checks (paste the output)
1. Generator prints the shape numbers above and the triangle count; two runs give identical bytes.
2. `--import` twice, no ERROR / WARNING lines.
3. Headless contract: node layout <...>, <n> bones with the names above, animations with lengths and loop modes, AABB
   within <...>.
4. Renders: side, top, three-quarter, close-up of <edge>, low side view for <gaps>; <in Godot preview | in Blender>.

Credits: <none (procedural) | source, author, licence per file>.
Waiting rule and One brief, one session rule: <paste>.
```

The lead reads the worker's "did not go as briefed" list first; a number the worker could not hold is a failed check.
