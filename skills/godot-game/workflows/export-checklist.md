# Export checklist (Godot 4, verified on 4.7.2 Windows Desktop)

## Preset

```ini
[preset.0]
name="Windows Desktop"
platform="Windows Desktop"
export_filter="all_resources"
include_filter="*.bin,*.md,*.txt,*.json"          # files read with FileAccess are not resources
exclude_filter="scenes/test/*,scenes/ui/ui_test.tscn"
export_path="../build/<Game>.exe"
```

and in `project.godot`:

```ini
[editor]
export/convert_text_resources_to_binary=false
```

(with `true`, exported `.tscn` files lost `PackedStringArray` `@export` values in 4.7.2; if you keep `true`, the smoke
test must check those values).

## Code rules that only bite in exports

- Never list resource folders with `DirAccess` (`get_files_at` returns `*.import` / remap names inside a `.pck`); use
  `ResourceLoader.list_directory(path)` and `ResourceLoader.exists(path)`.
- Never write `res://`. Never rely on files outside the project folder.
- `FileAccess` files (`.bin`, `.json`, `.md`, `.txt`, fonts licence text, credits) must match the include filter.
- `OS.has_feature("editor")` / `"template"` to branch dev-only code.

## Commands

```
<godot-exe> --headless --path <project> --export-release "Windows Desktop" ../build/<Game>.exe
<godot-exe> --headless --path <project> --export-pack "Windows Desktop" ../build/<Game>_test.pck
```

- `--export-release` / `--export-debug` need the export templates installed for that exact engine version.
- `--export-pack` writes only the `.pck`, handy for the smoke test.

## Smoke test of the exported data

The release template ignores `-s`, so run the exported pack with the **editor binary**:

```
<godot-exe> --headless --main-pack ../build/<Game>_test.pck -s <repo>/tools/export/smoke.gd -- <out.txt> exported
```

`smoke.gd` (kept outside `res://`) instantiates the main scene, starts the game, and checks with PASS / FAIL lines:
every `FileAccess` data file opens and has bytes; JSON files parse; counts that prove data loaded (terrain samples,
MultiMesh instance totals, credits lines, audio one-shots found through `ResourceLoader.list_directory`); dev scenes
absent (`not ResourceLoader.exists(...)`) only when run against the export; `quit(1)` on any FAIL.

Then the owner starts the real exe once by hand (window, input, audio, saves in `user://`).

## Before publishing (owner's word required)

- Version and product name in the preset; icon; console wrapper off or on as intended.
- Credits complete: every third-party asset with source, author, licence (CC BY with the full attribution line).
- The build folder is outside the repo or gitignored; `.pck` in Git LFS if it is ever committed.
