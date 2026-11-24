# Blender route (Blender MCP)

Use Blender for hand-shaped meshes, sculpts, retopology of scans, and when an artist will keep editing the asset. The
Blender MCP server runs Python in a live Blender with the full `bpy` API; in OpenCode it is often reached through a
code-mode `execute` tool (`tools.blender.*`). List the tools first.

## Practice

- One script per asset (`tools/models/<id>.py`) that builds the asset from scratch on a shared kit of helpers; the MCP
  call runs that script, so a rebuild is one call and the asset is reviewable as code. Never edit the kit or another
  asset's script in a parallel wave.
- Look up shader nodes by type, never by name (names are localised):
  `next(n for n in mat.node_tree.nodes if n.type == "BSDF_PRINCIPLED")`. Read enum identifiers from `bl_rna` instead of
  hard-coding them.
- Units: metric, scale 1.0, +Z up in Blender (the glTF exporter converts to +Y up). Apply transforms before export.
- Naming: objects as the game will look them up; actions named as the game's animation names (a `_loop` suffix
  gives a looping import, or patch loop modes after import).
- Materials: Principled BSDF with image textures (albedo, normal, ORM channels) exports cleanly to glTF PBR; procedural
  Blender nodes do not export: bake them to textures first.
- Render the brief's views in Blender for shape checks (top, low side, close-up of edges); judge shading in Godot with
  the game's materials and lighting.

## Export

`bpy.ops.export_scene.gltf(filepath=".../x.glb", export_format='GLB', use_selection=True, export_apply=True,
export_yup=True, export_animations=True, export_skins=True)` (check the exporter's parameter names for your Blender
version). Then the import patch and contract check as for generated assets (`gltf-import.md`).

## Library assets through Blender

Poly Haven models come as glTF at several resolutions (download directly, `sourcing-and-licences.md`); open them in
Blender only to edit (decimate, split LODs, re-pivot), then export again under the project's naming.
