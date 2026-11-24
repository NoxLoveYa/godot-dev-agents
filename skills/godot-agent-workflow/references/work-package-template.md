# Work package template

```
### WP-<X>: <title>

- Owns (new): <files this package creates>
- Owns (edits): <existing files only this package edits in this wave>
- Reads only: <files and contracts it depends on>
- Never edits: main scene, project.godot (except <keys>), Settings autoload, export_presets.cfg, CREDITS.md,
  <other packages' files>
- Depends on: <WP-A>; optional at runtime: <WP-B, WP-D>
- Steps: (1) <...> (2) <...>
- Acceptance:
  - Headless: `<godot-exe> --headless --path <project> [--fixed-fps 60] -s <tools>/<area>/test_<x>.gd` all PASS:
    <each check with its threshold>
  - Shots (windowed, cleared with the owner): `<shots runner> <name> "<args>" <frames>` vs <reference>:
    metrics `<check>` PASS + contact sheet + verdict per must-show item
  - Perf (only with an owner-approved budget): <delta ≤ n ms GPU at <preset> in <views>> by interleaved A/B
  - Existing tests still pass: <list>
- Credits: docs/credits_pending/<X>.md
- Report: files, check output, shots, measured numbers, notes for files it does not own, open items
```

## Cutting packages

- **By owner of state or look, not by layer:** "sky, key light and golden hour", "weather FX", "seasons, surfaces and
  ice", "living pond and fish AI", "audio, integration, performance". Each owns its scenes, scripts, shaders, tools,
  tests and assets.
- **One shared-file edit per wave** goes to one named package (e.g. the sky package may edit only the `[rendering]`
  keys of `project.godot`); everything else waits for integration.
- **Size:** one package = one brief that fits a session; a big package splits into ordered briefs (plants, then fauna).
- **Edits to another system's file** (a water shader patch) happen only after that system's own agent finished, in
  the contracts package, as a small listed patch.
- **Stubs first:** a shared include that the wave fills (`rain_ripples.gdshaderinc` returning zero) is created by the
  contracts package, and the owning package replaces it.

## Shared-file rules for the wave (copy into the design)

Each file has exactly one owner. Nobody edits the main scene, `project.godot` (except the named keys), the Settings
autoload or `CREDITS.md`. Credits go to `docs/credits_pending/<WP>.md` and the integration package merges them. A
package that needs a change in someone else's file writes it as a note in its report for the integration package.
