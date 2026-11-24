# Godot 4 command line for agents

`<godot-exe>` is the editor binary (the one with the editor tools). Paths in examples: `<project>` holds
`project.godot`; `<tools>` is the repo's `tools/` folder, outside `res://`.

## Flags used most

| Flag | Meaning |
|---|---|
| `--path <project>` | the project to run |
| `--headless` | no window, dummy audio and rendering; logic and data only |
| `-s <file.gd>` / `--script` | run a script that extends `SceneTree` (or `MainLoop`) instead of the main scene |
| `--check-only` | with `-s`: parse the script and quit (syntax check of one file) |
| `--import` | start the editor headless, import everything that changed, quit (creates `.import` files, `.godot/` cache) |
| `--fixed-fps <n>` | every frame advances exactly 1/n s; no real-time sync; deterministic physics with n = ticks per second |
| `--quit-after <n>` | quit after n frames |
| `--write-movie <file>` | Movie Maker: render every frame to `<file>` (`.png` sequence, `.avi` MJPEG; check the docs of your version for others); needs a window |
| `--resolution <w>x<h>` | window size (Movie Maker output size) |
| `--disable-vsync` | uncapped frame rate for frame-time runs |
| `--main-pack <file.pck>` | run an exported pack with this binary |
| `--export-release <preset> <path>`, `--export-debug`, `--export-pack` | builds (with `--headless`) |
| `--rendering-driver <d3d12\|vulkan\|opengl3>`, `--rendering-method <forward_plus\|mobile\|gl_compatibility>` | override the renderer |
| `--verbose` | more engine output |
| `-- <args>` | everything after `--` is for your script: `OS.get_cmdline_user_args()` |

`<godot-exe> --help` prints the full list for the installed version; read it once when a flag matters.

## The SceneTree script

```gdscript
extends SceneTree
## tools/<area>/test_thing.gd: headless test of <system>. Outside res://.
## Run: <godot-exe> --headless --path <project> --fixed-fps 60 -s <tools>/<area>/test_thing.gd [-- --quick]

var fails := 0
var checks := 0

func _initialize() -> void:
	_run()            # a coroutine: _initialize returns, the tree keeps running frames while _run awaits

func check(name: String, cond: bool, extra := "") -> void:
	checks += 1
	if not cond:
		fails += 1
	print("  %s  %s%s" % ["PASS" if cond else "FAIL", name, ("  (" + extra + ")") if extra != "" else ""])

func _run() -> void:
	var settings := root.get_node("Settings")          # autoloads exist; reach them by node name
	settings.set("cfg_path", "user://settings_test.cfg")
	var main := (load("res://scenes/main.tscn") as PackedScene).instantiate()
	root.add_child(main)
	for i in 30:
		await process_frame
	check("water exists", get_first_node_in_group("water") != null)
	print("== summary: %d checks, %d failed" % [checks, fails])
	main.queue_free()
	await process_frame
	quit(1 if fails > 0 else 0)
```

- `_initialize()` runs once; `_process(delta) -> bool` runs every frame (return `true` to quit) if you prefer a frame
  counter to a coroutine.
- The script is the tree: `root`, `process_frame`, `physics_frame`, `create_timer`, `get_first_node_in_group` are
  members.
- `load("<absolute path>/helper.gd")` loads helpers that live outside `res://` (shared argument parsing, camera
  placement). Godot accepts absolute paths for `load` in the editor binary.
- Exit code: `quit(code)`; the runner checks it and also scans the log for `FAIL` and engine errors.

## Reading the log

- Engine errors: `SCRIPT ERROR`, `Parse Error`, `ERROR:`, `WARNING:`.
- Ignore the lines a headless run prints at exit about resources "still in use at exit" / ObjectDB instances "leaked
  at exit" unless you are hunting leaks.
- Strip ANSI colour codes before grepping (`-replace '\x1b\[[0-9;]*m', ''`).

## Exported builds

- The release template ignores `-s`; test a `.pck` with the editor binary and `--main-pack`
  (`godot-game/workflows/export-checklist.md`).
