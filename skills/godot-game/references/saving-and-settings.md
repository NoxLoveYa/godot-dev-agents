# Settings, saves and user://

## Where files go

- `user://` is the per-user data folder (on Windows under `%APPDATA%`, per project name, or a custom user dir when
  `application/config/use_custom_user_dir` is set). `res://` is read-only in an export: never write it at runtime.
- `ProjectSettings.globalize_path("user://x.cfg")` gives the absolute path (for logs and tools).

## ConfigFile for settings and small state

```gdscript
const PATH := "user://settings.cfg"
var cfg_path := PATH            # tests point this at a temp file

func save() -> void:
	var cf := ConfigFile.new()
	cf.set_value("meta", "version", 1)
	for k: StringName in DEFS:
		cf.set_value(DEFS[k]["sec"], String(k), _v[k])
	var err := cf.save(cfg_path)
	if err != OK:
		push_warning("Settings: cannot write %s (%d)" % [cfg_path, err])

func load_from_disk() -> bool:
	var cf := ConfigFile.new()
	var ok := cf.load(cfg_path) == OK
	for k: StringName in DEFS:
		var d: Variant = DEFS[k]["def"]
		_v[k] = _coerce(k, cf.get_value(DEFS[k]["sec"], String(k), d)) if ok else d
	return ok
```

- **Validate everything on load**: type, clamp to `min`/`max`, allowed `values`, enum bounds. An unreadable file or
  an unknown version falls back to defaults and is overwritten on the next save.
- Save on BACK from the options page, on quit (the state machine's quit path) and on
  `NOTIFICATION_WM_CLOSE_REQUEST`.
- A `version` key from day one; migrations read old keys and write new ones.

## Presets and live apply

- A preset is a dictionary of values for `PRESET_KEYS`; picking it sets them all (guard flag so each set does not
  flip the preset to CUSTOM); a manual change of one key sets CUSTOM.
- Keys that need the world (Environment flags, shadow mode, vegetation scaling) are applied to the first matching
  node found and re-applied when such a node enters the tree (`get_tree().node_added`).
- Window mode and resolution: apply on change; do not override a window the user resized unless they touched the
  setting.
- Mute when unfocused: `NOTIFICATION_APPLICATION_FOCUS_OUT` / `FOCUS_IN`.

## Saves of game state

- World state that may be remembered (clock, weather, RNG state) goes in its own file (`user://environment.cfg`),
  owned by its state owner, behind an opt-in flag when "always start at the default moment" is the design.
- Large or structured saves: `FileAccess` + `var_to_bytes` / JSON with a version field; write to a temp file and
  rename, so a crash mid-write does not corrupt the save.
- RNG state (`rng.state`) is part of the save when replays or determinism matter.

## Tests and tools (hard rule)

- Every persistent object has a path override (`cfg_path`, `save_path`) and a persist switch (`persist = false`).
  Tests set both before the object loads, use throw-away instances (`AppClimate.new()` not in the tree) where they can,
  and delete their temp files at the end.
- A tool that must change a setting for a measurement (vsync off) uses a temp config path, never renames or deletes
  the player's file.
- Headless tests that load the main scene still create the autoloads, which load the real `user://` files at
  startup: such tests must not save. Flip `persist` off first thing in `_initialize`.
