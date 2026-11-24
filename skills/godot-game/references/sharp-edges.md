# Sharp edges

Each row cost the reference project a pass. **(4.7.2)** = verified on Godot 4.7.2, Windows, Forward+, d3d12.

## Process and tests

| Issue | What happened | Rule |
|---|---|---|
| A test wrote into `res://` | a test once overwrote and then deleted a real scene | tests and tools never write `res://`; outputs go to `tools/<area>/out/` |
| Tests wrote the player's `user://` files | settings and world state of the real player were replaced | path override + persist flag on every persistent object; temp files deleted at the end |
| `-s` scripts and autoload names **(4.7.2)** | a test script that names an autoload (`Settings.get_value`) statically does not compile reliably | `root.get_node("Settings")`; inside game code prefer `AppSettings.get_instance()` |
| Non-deterministic behaviour tests | fish and fauna tests passed or failed by run | `--fixed-fps 60`; seed every RNG **before** deferred spawns; a deterministic flag on spawners |
| Time does not run in a headless main-scene test | systems that advance only while PLAYING stayed still | start the game through the state machine first |
| Parallel agents in one working tree | lost edits when two agents wrote one file | exact file ownership per package; notes for files you do not own |
| Shared `user://` between headless tests | tests run in parallel clobbered each other's files | run the regression set serially |

## Windows and the CLI

| Issue | Rule |
|---|---|
| Windowed Godot runs steal focus from the owner's fullscreen app; external window-guard scripts and project no-focus overrides did not prevent it | prefer `--headless`; batch windowed runs; tell the owner before them |
| `godot.exe` is a GUI-subsystem app: PowerShell does not wait for it and drops its output | pipe or redirect output (`2>&1 \| Out-File`, or `Start-Process -Wait -Redirect...`), or use the console wrapper exe if your install has one |
| A plain pipeline sometimes hung on a headless run | redirect to a log file with a timeout (`Start-Process ... -PassThru`, then wait on the PID) |
| `powershell -File script.ps1 "--arg"` drops quoted args starting with `--` | call scripts as `& .\script.ps1 ...` from PowerShell |
| `-Only a,b` through `-File` arrives as one string | split on `,` inside the script |
| Movie Maker writes nothing when the output folder is missing | create the folder first |
| The release export template ignores `-s` | smoke-test with the editor binary: `--main-pack <game>.pck -s <smoke.gd>` |

## Export **(4.7.2)**

| Issue | Rule |
|---|---|
| Files read with `FileAccess` (`.bin`, `.json`, `.md`, `.txt`) are not resources and were skipped by the export | `include_filter="*.bin,*.json,*.md,*.txt"` (or narrower) in the preset |
| `DirAccess.get_files_at` inside a `.pck` lists `*.import` / remap names | `ResourceLoader.list_directory()` |
| With `editor/export/convert_text_resources_to_binary=true`, exported `.tscn` lost `PackedStringArray` `@export` values | set it `false`, or verify the values in the smoke test |
| Dev scenes shipped | `exclude_filter` + a smoke check that they are absent |

## Rendering **(4.7.2)**

| Issue | Rule |
|---|---|
| Visibility-range fade margins draw two LODs at once; in a dense forest that cost 5-9 ms | short margins (or fade disabled) on dense layers; scale them with the quality preset |
| SDFGI cascades caused walk spikes | 4 -> 2 cascades removed most spikes; far cascades only lit fogged distance |
| The directional shadow pass ran in a storm where shadows were invisible | gate `shadow_enabled` off when key energy x shadow opacity is tiny, with an opacity fade so nothing pops |
| A sky shader that uses `TIME` re-renders the radiance cubemap every frame | amortise (octahedral cloud map, a few texels per frame) and use `Sky.PROCESS_MODE_INCREMENTAL` |
| GPUParticles collision sub-emitters did not spawn | kill particles at a height texture / collision instead and spawn splashes another way |
| Alpha-blended materials vanished behind screen-texture refraction | alpha scissor / hash (opaque pipeline) for anything seen through refractive water |
| Perf "win" from `cast_shadow = OFF` on a shadow-only proxy | that makes the proxy visible (a black occluder hiding geometry): measure by hiding (`visible = false`) |
| A per-frame measurement on a shared GPU | interleaved A/B blocks in one process, min / median of block means |

## Engine behaviour worth knowing

| Issue | Rule |
|---|---|
| Imported meshes are shared resources | `duplicate()` before setting materials per builder, or one builder's materials replace another's |
| `add_child` from a parent's `_ready` | `call_deferred` |
| A paused tree freezes menus | `PROCESS_MODE_ALWAYS` on menus and on whatever the pause menu previews |
| A UI page closed but still eating clicks | `mouse_filter = IGNORE` when closed |
| Built-in `ui_accept` / `ui_cancel` had no pad buttons **(4.7.2, this project's map)** | add A / B at runtime when missing |
| SubViewports set to `UPDATE_ONCE` rendered in creation order **(4.7.2, probed)** | build multi-pass chains front to back; probe it if you rely on it |
