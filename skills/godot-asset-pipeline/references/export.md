# Assets in exports

Full checklist: `../../godot-game/workflows/export-checklist.md`. The asset-side rules:

- **Data files read with `FileAccess`** (`.bin` heightmaps, scatter, LUTs, noise volumes; `.json` manifests; `.md`
  credits; `.txt` font licences) are not resources: the export skips them unless the preset's `include_filter` names
  them (`*.bin,*.json,*.md,*.txt`) **(4.7.2)**.
- **Listing assets at runtime:** `ResourceLoader.list_directory(dir)` (lists loadable resources in editor and export);
  `DirAccess.get_files_at` inside a `.pck` lists `*.import` / remap names **(4.7.2)**. Keep a manifest (`manifest.json`
  from the generator) when order or metadata matters.
- **Text vs binary resources:** with `editor/export/convert_text_resources_to_binary=true`, exported `.tscn` lost
  `PackedStringArray` `@export` values in 4.7.2; set it `false` or check those values in the smoke test.
- **Dev assets** (test scenes, previews) in `exclude_filter`; the smoke test checks they are absent from the export.
- **Smoke test** (`tools/export/smoke.gd`, run with the editor binary and `--main-pack <game>.pck`): every data file
  opens with bytes, JSON parses, counts prove the data loaded (terrain samples, MultiMesh instances, credits lines,
  audio one-shots via `list_directory`).
