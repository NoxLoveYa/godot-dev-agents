---
name: godot-asset-pipeline
description: "Assets into Godot 4: glTF scene import options, patching .glb.import (animation loop modes, LOD generation, extracted texture compression), animation and node naming, procedural model generators in Python/numpy writing glTF with skeleton, skin and animations, the Blender MCP route, CC0 sourcing through the Poly Haven API and others, licences and credits, Git LFS, export include filters and the export smoke test. Use when a design needs models, textures, sounds or data files, or an export drops them."
---

# Asset pipeline (Godot 4)

## Rules

1. **Route per asset:** a CC0 library asset when one fits; a procedural Python generator for organic or parametric
   shapes, families of variants and anything that must be regenerated with new numbers; Blender for hand-shaped meshes
   and sculpts. Write the route and why into the brief (`workflows/asset-brief.md`).
2. **Generators are deterministic** (fixed seeds, byte-identical reruns), parametric (numbers as constants at the top),
   and own their output files; nobody hand-edits generated files.
3. **Import settings are code:** `.glb.import` patches applied by a script after the first import, then a second import
   (`references/gltf-import.md`). Never rely on clicks in the import dock that nobody can repeat.
4. **Every asset has a contract** checked headless (node names, surfaces, bone count and names, animation names /
   lengths / loop modes, AABB, triangle count) and a render the lead looks at.
5. **Licences before use:** source URL, author, licence, edits recorded per file; CC0 / PD first; CC BY with the full
   attribution line; never NC / ND / unknown (`references/sourcing-and-licences.md`).
6. **Binaries in Git LFS**; raw downloads outside the project and gitignored.
7. **Exports carry data files** (`include_filter`) and pass the smoke test (`references/export.md`).

## Routing

| The task | Load |
|---|---|
| Import options, `.import` patching, loop modes, LODs, texture compression, naming suffixes | `references/gltf-import.md` |
| Writing a generator: SDF / marching cubes / lathe, UVs, baked textures, skeleton, skin, animations, GLB writer | `references/procedural-generators.md` |
| Building in Blender through its MCP, export settings | `references/blender-route.md` |
| Poly Haven / ambientCG / OpenGameArt / Internet Archive, licences, credits, Git LFS | `references/sourcing-and-licences.md` |
| Export filters, `ResourceLoader.list_directory`, smoke test | `references/export.md` |
| Writing a worker brief | `workflows/asset-brief.md` |
| Regenerate and re-import an asset | `workflows/regen-and-import.md` |
