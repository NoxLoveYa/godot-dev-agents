# Procedural model generators (Python + numpy -> glTF)

Used in the reference project for four fish species (skinned, swim / idle / bite animations) and a frog (SDF body,
webbed digits, 18-bone rig, four animations), and for trees, plants and pond scatter. Each generator takes about a
minute, is deterministic, and is re-run whenever the design's numbers change.

## Shape

| Method | When | Tools |
|---|---|---|
| **Lathe / loft along a spine** | fish bodies, stems, trunks, leaves: cross-sections (ellipses or profiles) along a curve, radii from a PCHIP-interpolated profile table | numpy |
| **SDF + marching cubes** | organic bodies with limbs (a frog): smooth-union of capsules / ellipsoids in millimetres, mesh with marching cubes, Taubin smoothing, details pushed out as geometry, quadric decimation to the budget | numpy, scipy, scikit-image (`measure.marching_cubes`), fast_simplification |
| **Cards and clumps** | grass, ribbon weed, ferns: alpha-tested quads with vertex colour masks for wind (flex, phase, height, tip) | numpy, Pillow |
| **Scatter** | placement layers: rejection sampling on masks (depth bands, slope, shelter, distance to shore), deterministic RNG per layer | numpy |

Shape checks print the numbers the brief fixes (length, height, girth, fin spans, triangle count) from the vertices.

## UVs and textures

- Unwrap with xatlas (`pip install xatlas`) or analytic UVs for lathes (u around, v along).
- **Paint as functions of the 3D position** (blotches, stripes, counter-shading, belly colour) and bake them into the UV
  atlas: no seams, any resolution, deterministic. Normal map from a procedural relief (granular skin, scales) baked the
  same way; ORM with per-texel ambient occlusion (from the SDF or ray casts on the mesh).
- Separate materials for parts with different needs (body double-sided for webbing, eyes glossy, a throat sac).

## Skeleton, skin, animation

- Bones as a chain or tree with names the game uses (`root spine head ...`, `leg_l_thigh ...`); inverse bind matrices
  from the bind pose.
- **Soft, distance-based weights** (a few nearest bones by distance along the spine or to bone segments, smooth
  falloff, normalised to 4 influences) avoid segment seams.
- Animations as glTF channels (rotation quaternions, translation) sampled at 30 fps: a swim cycle as a travelling wave
  down the spine (amplitude growing toward the tail), idle as a small version, one-shots (bite, croak, jump). Name
  them as the game expects; loop modes come from the import patch.

## Writing the GLB

A minimal writer (~70 lines) is enough: one binary buffer, buffer views aligned to 4 bytes, accessors with min / max
for positions, meshes with `POSITION NORMAL TANGENT TEXCOORD_0 JOINTS_0 WEIGHTS_0 COLOR_0`, materials with PBR
textures (PNG bytes embedded as images), nodes, one skin, animations, one scene. Component types: FLOAT 5126,
UNSIGNED_SHORT 5123, UNSIGNED_BYTE 5121, UNSIGNED_INT 5125. Validate a new writer once with the Khronos glTF validator
**(unverified for your writer: run it)**.

## Pipeline script (`regen.ps1`)

1. Run the generator (`python -I generate_<x>.py [variants]`).
2. Delete the stale extracted textures of the previous output (`<name>_*.png*`).
3. `<godot-exe> --headless --path <project> --import`.
4. `python -I patch_import.py` (loop modes, LODs off, texture compression).
5. `--import` again; once more, printing only `ERROR|WARNING` lines (should print nothing).
6. The headless contract check; preview renders (windowed, when allowed).

## Determinism and speed

- One seeded `np.random.default_rng(seed)` per part; never global RNG state.
- Vectorise with numpy; independent variants in parallel processes (cores - 2), never threads.
- Two runs, identical bytes: checked by a hash in the generator's test.
