---
name: godot-cli-testing
description: "Drive Godot 4 from the command line as an agent: headless -s test scripts with PASS/FAIL assertions, --fixed-fps determinism, Movie Maker screenshots and contact sheets, numeric metrics against reference images, interleaved GPU perf A/B, soak tests, a serial regression runner, the windowed-run focus warning and PowerShell gotchas. Use whenever an agent runs, tests or captures a Godot project."
---

# Godot from the command line

Godot has no official MCP: an agent runs it as a program. Three kinds of run:

| Run | Flags | Window | Use |
|---|---|---|---|
| **Headless test** | `--headless --path <project> [--fixed-fps 60] -s <tools>/test_x.gd [-- args]` | none | logic, contracts, data, behaviour; the default check |
| **Import / export** | `--headless --path <project> --import` / `--export-release` / `--export-pack` | none | after new assets or scripts; builds |
| **Windowed run** | no `--headless`: `--write-movie`, perf probes, GPU-only behaviour | **yes, focusable** | screenshots, video, frame time |

Commands, flags and the script shape: `references/cli-reference.md`.

## Rules

1. **Headless first.** Everything that is not a picture or a frame time is a headless test. A long behaviour runs in a
   headless script with `--fixed-fps 60` (simulated time runs faster than real time).
2. **Windowed runs steal focus.** Every windowed Godot run opens a focusable window that can pull the owner out of a
   fullscreen game; external window-guard scripts and project no-focus settings did not stop it. Run windowed only when
   the owner asked or the brief's result is a picture, batch the runs, and say how many you ran. One windowed Godot at
   a time; on a GPU shared by several agents expect about one frame per minute in the worst case, so size shot counts
   and frame counts to it.
3. **Tests never write `res://` content or the player's `user://` files.** Temp paths, `persist = false`, throw-away
   instances, outputs in `tools/<area>/out/` (gitignored). A test once overwrote and deleted a real scene.
4. **Determinism:** `--fixed-fps 60` for anything that simulates time; seed every RNG before deferred spawns; wait in
   frames (`await process_frame`) or physics frames, never wall-clock seconds.
5. **Autoloads in `-s` scripts:** reach them with `root.get_node("Name")`; never use the autoload's global identifier
   statically in a `-s` script.
6. **Small results.** Print `PASS` / `FAIL` lines and one summary line; exit code 1 on any failure. Report counts and the
   failing lines, not whole logs.
7. **Stop only processes you started**, by PID.

## Routing

| The task | Load |
|---|---|
| Flags, the SceneTree script shape, user args, exit codes | `references/cli-reference.md` |
| Writing a headless test (assertions, fixtures, main-scene tests, determinism) | `references/headless-tests.md` |
| Screenshots with Movie Maker, the shot harness, named cameras, video frames | `references/movie-maker-shots.md` |
| Contact sheets, numeric metrics against reference images, written verdicts | `references/metrics-and-references.md` |
| GPU / CPU timing, interleaved A/B on a shared GPU | `references/perf-ab.md` (more in the `godot-performance` skill) |
| Soak test, regression runner | `workflows/soak-and-regression.md` |
| PowerShell and Windows traps | `references/windows-powershell.md` |

## Minimal check (what a coder runs by default)

1. `<godot-exe> --headless --path <project> --import` after new scripts, `class_name` changes or assets; grep the
   output for `ERROR|SCRIPT ERROR|Parse Error`.
2. The area's headless tests once (or the regression runner with `-Only <names>`).
3. Nothing windowed unless the brief allows it.
