# New Godot project

1. **Profile in one page** (owner's words): genre, camera, platforms and input (keyboard / pad / touch), target
   resolution and fps per platform, renderer (Forward+ for desktop realism, Mobile, Compatibility for web / old GPUs),
   look references, scope of the first playable. Write it into `docs/game-profile.md`.
2. **Repo layout** (`references/project-layout.md`): `<project>/` for Godot, `tools/`, `docs/design/`, `AGENTS.md` from
   `templates/AGENTS.md`. `.gitignore`: `.godot/`, `/build/`, `/tools/**/out/`, raw downloads. `.gitattributes` with Git
   LFS for binaries (`godot-asset-pipeline/references/sourcing-and-licences.md`).
3. **project.godot first pass**: `config/features` (engine version, renderer), stretch mode `canvas_items` / `expand`,
   physics engine (Jolt for 3D), rendering driver, input actions with keyboard and pad events, an empty
   `[shader_globals]` with the first contracts (wind, player position, water level as needed).
4. **Autoloads**: `Settings` (DEFS + presets + ConfigFile), `GameFlow` (state machine), plus state owners the design
   needs. Static accessors and duplicate guards (`references/architecture-patterns.md`).
5. **Main scene skeleton**: WorldEnvironment, the key light, the player, a UI root, a module loader node.
6. **Tooling before content** (`godot-cli-testing`): a headless test runner, one smoke test that loads the main scene
   and starts the game, a shot harness with named cameras, a regression runner script, a `tools/README.md`.
7. **Export preset** early (`workflows/export-checklist.md`), so export problems show while the project is small.
8. **Design docs and packages** for the first systems (`godot-agent-workflow`).

Check: the main scene boots headless with a clean log; the smoke test passes; an export `.pck` passes the smoke test
with `--main-pack`.
