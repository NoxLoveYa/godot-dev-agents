# Movie Maker screenshots and frames

Movie Maker (`--write-movie`) renders every frame at a fixed time step regardless of GPU speed, so a screenshot or a
clip is deterministic and complete even when the GPU is shared. It needs a window: every shot is a **windowed run**
(focus warning, `../SKILL.md` rule 2).

## Command

```
<godot-exe> --path <project> --resolution 1920x1080 --write-movie <out>/_mm_<name>/f.png --fixed-fps 30 --quit-after 120 -s <tools>/shot_env.gd -- --cam=spawn --hour=19.5 --snap --frames=120
```

- **Create the output folder first**; Movie Maker does not.
- PNG sequences are numbered (`f00000000.png` ...); keep the last frame as the shot, delete the rest. A `.wav` of the
  audio is written next to the frames.
- Frames: 90+ before the shot frame so SDFGI, TAA and temporal volumetric fog settle (120 is a safe default; more for
  a scene that streams or fades in).
- Expect long runs on a shared GPU (in the worst case about one frame per minute with several agents rendering):
  batch shots in one runner script and run them back to back.

## The shot harness (kept outside `res://`)

A `SceneTree` script that:

1. parses arguments (`--cam=<name | x,y,z,yaw,pitch>`, world state such as `--hour= --season= --weather=`,
   overrides `--off=<modules>`, `--quality=`, `--env=prop:value`, `--setting=key:value`, `--fov=`, `--frames=`), shared
   with the perf tools through one `common.gd` loaded by absolute path;
2. instantiates the main scene, optionally skipping modules before it enters the tree;
3. on frame 2 places the camera (named cameras in `cams.json`: position, yaw, pitch, fov), forces the world state and
   freezes the clock, so the picture is deterministic;
4. applies overrides on a later frame (after modules finished their own setup, e.g. frame 40);
5. calls the look owner's `snap()` right after forcing state and again just before the last frame, so smoothing does
   not leave the look halfway;
6. prints one `STATE` line near the end (clock, weather, sun elevation, key values) for the report.

A PowerShell runner per area (`shots.ps1 <name> "<args>" [frames]`) creates the folder, runs Godot with the output
piped (so the shell waits), keeps the last frame as `<out>/<name>.png`, prints the `ERROR|WARNING|SCRIPT|STATE` lines,
and removes the frame folder.

## Cheaper shots

- A **sky-only** or **material-only** scene (the look module on an empty scene with the main Environment) renders in
  seconds instead of minutes: tune there, confirm in the full scene.
- A **dev override** argument read by the look owner (`--look=key=value,...`, raw shader uniforms `u_<name>=v`) lets you
  try values without editing defaults.
- Time-lapses: one process per hour list, a contact sheet of the frames, numbers per frame (`metrics-and-references.md`).

## Clips

- `--write-movie <out>/clip.avi` (MJPEG) or a PNG sequence encoded with ffmpeg:
  `ffmpeg -framerate 30 -i f%08d.png -i f.wav -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a aac clip.mp4`.
- A moving-sun clip at 10x day speed shows shadow crawl and pops; analyse frame-to-frame differences numerically.

## Asset previews

A preview scene for one model (turntable or fixed views: side, top, three-quarter, close-up), shot with the same
harness, is the Godot-side render check for assets (the materials and lighting the game uses).
