# glTF import in Godot 4

## Conventions

- glTF is metres, +Y up. Godot's forward is -Z. Pick one facing for the project (the reference project: heads toward
  -Z, origin on the ground under the body) and write it in every brief; check it with a top-view render.
- `.glb` (binary, textures embedded) is simplest for generated assets. Godot extracts embedded textures to
  `<name>_<image>.png` files next to the `.glb` on import, each with its own `.png.import`.

## Import options (`<file>.glb.import`, `[params]`)

| Key | Use |
|---|---|
| `nodes/root_type`, `nodes/root_name` | root node type / name of the imported scene |
| `meshes/generate_lods` | engine auto-LOD; turn **off** for skinned characters and hand-made LOD sets |
| `meshes/create_shadow_meshes` | shadow-only simplified meshes (on by default) |
| `meshes/light_baking` | static / dynamic / disabled for GI |
| `skins/use_named_skins` | keep skin names |
| `animation/import`, `animation/fps`, `animation/trimming` | animation import |
| `_subresources` | per-animation, per-mesh, per-material overrides (a JSON-like dictionary) |

Loop modes per animation in `_subresources` (0 none, 1 linear, 2 ping-pong):

```ini
_subresources={"animations":{"swim":{"settings/loop_mode":1},"idle":{"settings/loop_mode":1},"bite":{"settings/loop_mode":0}}}
```

## Patching after the first import

`.import` files exist only after Godot has imported the file once. So: generate -> `--import` -> patch -> `--import`
again (applies the new options) -> (once more if extracted textures were regenerated, to settle their uids).

```python
# patch_import.py (idempotent): loop modes, no generated LODs
import re
SUB = '_subresources={"animations":{"idle":{"settings/loop_mode":1},"swim":{"settings/loop_mode":1},"jump":{"settings/loop_mode":0}}}'
txt = open(p, encoding="utf8").read()
new = re.sub(r"^_subresources=.*?(?=^gltf/|\Z)", SUB + "\n", txt, flags=re.M | re.S)
new = re.sub(r"^meshes/generate_lods=.*$", "meshes/generate_lods=false", new, flags=re.M)
if new != txt:
    open(p, "w", encoding="utf8", newline="\n").write(new)
```

Extracted textures (`<name>_*.png.import`):

```python
n = re.sub(r"^compress/mode=.*$", "compress/mode=2", t, flags=re.M)            # 2 = VRAM compressed
n = re.sub(r"^compress/high_quality=.*$", "compress/high_quality=true", n, flags=re.M)   # BC7 on desktop
if "normal" in name:
    n = re.sub(r"^compress/normal_map=.*$", "compress/normal_map=1", n, flags=re.M)
```

Small textures that must stay exact (eyes, UI) stay lossless. Delete stale extracted textures of the previous version
before re-importing a regenerated `.glb`.

## Naming that the importer understands

- Animation names ending in `-loop` / `_loop` or `-cycle` / `_cycle` import as looping (an alternative to patching).
- Node name suffixes: `-col` (mesh + collision), `-colonly` (collision only), `-convcolonly` (convex collision only),
  `-noimp` (skip), `-rigid`, `-navmesh`, `-occ` (occluder). Check the docs of your version before relying on one.

## After import

- Imported meshes are **shared resources**: code that sets materials on them works on `duplicate()`s.
- Inspect a model headless: instantiate the `.glb` as a `PackedScene`, walk nodes, print `MeshInstance3D` surfaces and
  triangle counts, the `Skeleton3D` bone names, the `AnimationPlayer` animations with `length` and `loop_mode`, the
  AABB. A contract test asserts them.
- Texture budgets: 2k albedo / normal / ORM for a hero creature at close range, 1k or less for props; VRAM compressed;
  mipmaps on; normal maps flagged.
