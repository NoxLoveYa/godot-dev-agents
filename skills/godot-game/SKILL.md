---
name: godot-game
description: "Godot 4 hub: project layout, scenes, nodes, signals, autoloads, resources, GDScript style and typing, input map, UI and gamepad focus, saving with ConfigFile, audio buses, Jolt physics, architecture patterns (contracts, global shader uniforms, module scenes, GameFlow, Settings presets), export; routes to the other Godot skills. Use for any Godot or GDScript task."
user-invocable: true
---

# Godot game development hub

Godot 4.x knowledge (`references/`), guided workflows (`workflows/`) and a project rules skeleton (`templates/`). Facts
marked **(4.7.2)** were verified on Godot 4.7.2 (Windows, Forward+, d3d12) in a realistic forest-pond game (the
reference project); other versions may differ. Items marked **(unverified)** are believed true but were not tested.

## Before anything

1. **Read the project's `AGENTS.md`** (skeleton: `templates/AGENTS.md`) and its design docs. Engine version, renderer,
   platforms, budgets, ownership tables and paths (`<godot-exe>`, `<project>`) come from there.
2. **Read `project.godot`**: `config/features` (engine version, renderer), `[autoload]`, `[shader_globals]`,
   `[input]`, `[rendering]`. Those sections are the project's contracts.
3. **House rules win** over the generic code in the references (end of this file).

## Godot has no official MCP

Agents drive Godot through the command line: headless `-s` scripts for logic and checks, `--import`, `--export-*`,
Movie Maker for pictures. Everything about running Godot from an agent is in the `godot-cli-testing` skill. Blender
has an MCP; see `godot-asset-pipeline`.

## Routing

| The task | Load |
|---|---|
| Plan a feature or a whole system across agents: design doc, work packages, briefs | the `godot-agent-workflow` skill |
| Start a new project | `workflows/new-project.md`, `templates/AGENTS.md` |
| Folders, scenes, nodes, signals, autoloads, resources, groups, process order | `references/project-layout.md` |
| GDScript style, static typing, doc comments, per-frame code | `references/gdscript-style.md` |
| How modules talk: contracts, global shader uniforms, module scenes, state machines, ownership of engine fields | `references/architecture-patterns.md` |
| Menus, HUD, themes, focus, gamepad, input map, rebinding | `references/ui-and-input.md` |
| Settings, presets, save files, `user://` | `references/saving-and-settings.md` |
| Buses, players, ambience, import settings for sound | `references/audio.md`; sourcing in `godot-asset-pipeline` |
| CharacterBody3D, collision layers, raycasts, terrain collision, Jolt | `references/physics.md` |
| Headless tests, Movie Maker shots, metrics, perf A/B, soak, regression, PowerShell | the `godot-cli-testing` skill |
| Environment, sky, clouds, water, foliage, weather, seasons, matching a reference | the `godot-rendering-look` skill |
| Frame time, budgets, quality presets, profiling, LOD, CPU throttling | the `godot-performance` skill |
| glTF import, procedural models, Blender, Poly Haven, licences, Git LFS | the `godot-asset-pipeline` skill |
| Export presets, include filters, the exported-build smoke test | `workflows/export-checklist.md` |
| Fix a bug, an error in the log | `workflows/debug-loop.md` |
| Review code | `workflows/code-review.md` |
| "Why does this break" | `references/sharp-edges.md` |

If the intent is ambiguous, ask one question, then route.

## Top sharp edges (full list: `references/sharp-edges.md`)

| Severity | Issue | Fix |
|---|---|---|
| CRITICAL | A test or tool overwrote and deleted a real scene in `res://` | Tests and tools never write `res://`; outputs go to a folder outside the project |
| CRITICAL | Tests overwrite the player's `user://` settings and saves | Every persistent singleton has a path override and a persist flag; tests set both |
| HIGH | Export ships without the `.bin` / `.json` / `.md` / `.txt` files read by `FileAccess` | `include_filter` in the preset; smoke-test the exported `.pck` |
| HIGH | Two modules write the same Environment field or global uniform | An ownership table in the design; one writer per field |
| HIGH | `-s` test scripts fail to compile on autoload names | `root.get_node("Name")`, never the global identifier |
| HIGH | Windowed Godot runs steal focus from the owner's fullscreen app | Prefer `--headless`; batch and announce windowed runs |

## House rules (override the references)

- **One writer per fact.** Every engine field, global uniform, file and autoload property has one owner, written in
  the design's ownership table. Readers duck-type (`has_method`, `"prop" in node`, groups), so modules work alone.
- **Contracts before code.** APIs, node paths, groups, signals, data formats and `[shader_globals]` are written and
  merged before parallel work starts; every global has a default so shaders compile before their writer exists.
- **Optional modules are scenes** loaded by a loader node if their file exists, so parallel packages never edit the
  main scene and the game degrades gracefully when a module is missing.
- **Content and tuning live in tables** (`const` dictionaries or resources at the top of the owning script, or data
  files from a generator), one source of truth per number.
- **Generated content is deterministic** (fixed seeds, byte-identical reruns) and regenerated, never hand-edited.
- **The owner tests by hand.** The agent's check is minimal (parses, clean log, the existing headless tests once).
  Windowed runs, shots and perf only when asked, or when the work's result is a picture.
- **Ask before anything outward-facing:** publishing builds, store pages, uploads, paid assets, data wipes.
