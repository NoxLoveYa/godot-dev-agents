# Worked example: atmosphere, weather, seasons and a living pond

A realistic first-person forest-pond game in Godot 4.7.2 (Forward+, Jolt, d3d12, 1080p High at 60+ fps on a shared
desktop GPU). The owner asked for a day / night cycle with a golden-hour highlight, eight weathers with smooth
transitions, four seasons with winter ice, an in-pause ENVIRONMENT panel, a living pond (plants, insects, fry, frogs,
fish that hide in plant beds), CC0 assets, and a shallow-water look matched to a reference photo.

## Recommendation (as approved)

1. One state owner (`Climate` autoload: clock, astronomy, weather chain, integrators, persistence, globals) and one
   look owner (`Atmosphere` module scene: sky, key light, fog, glow, exposure, grade).
2. Sky: a custom sky shader on a baked physical atmosphere LUT, amortised volumetric clouds; rejected
   PhysicalSkyMaterial (no clouds or weather) and HDRI blending (fixed sun positions, ghosting, 0.5-1 GB VRAM).
3. One key light: sun by day, moon by night.
4. Golden hour as a look target keyed by sun elevation, not a special mode.
5. Weather as a Metropolis-Hastings chain on a neighbour graph with staged transitions.
6. The living pond as baked deterministic scatter plus a small fauna module; fish use beds as cover.
7. Packages: WP-0 base integration -> WP-A contracts -> parallel WP-B, C, D, E -> WP-F integration.

## Constraints found in the project (excerpt)

- Existing globals `wind_direction`, `wind_strength`, `wind_speed`, `water_level`, `player_position`.
- The water node toggles `fog_enabled` / `volumetric_fog_enabled` underwater; Settings forces some flags at priority
  1000: **nobody else may write these**.
- Settings takes the first `DirectionalLight3D` child of the world root as the sun: extra lights must not be direct
  children of Main.
- Settings scales vegetation through builder nodes that are direct children of Main.
- No git at the time: parallel agents shared one working tree, so file ownership had to be strictly disjoint.

## Packages

| WP | Owns (excerpt) | Acceptance (excerpt) |
|---|---|---|
| A: contracts, Climate, panel, tooling | climate autoload, astronomy, weather table, module loader, env globals include, stub includes, ENVIRONMENT panel, all test / shot / metrics tools; patches water, vegetation, wind, UI root | headless: summer 19:30 elevation 6-9°, noon 58-61°, 24-min day `advance(60)` = +1.00 h ± 0.01, 200 simulated days per season within ±5 points of the share table, integrators reach targets, save / load round-trip, corrupt file -> defaults; a no-op shot equal to the pre-change render except the intended shallow tint |
| B: sky, key light, golden hour | atmosphere scene and script, look table, sky / capture / grade shaders, LUT baker, cloud noise generator, sky assets, look test | headless: key energy monotonic -2°..35°, moon mode at 01:00, golden key R/B ≥ 1.8, night EV +1.3..+1.8; shots vs the golden-hour and waterline references with metrics; a 10-hour time-lapse without pops; sky + capture + grade + SDFGI delta ≤ 0.9 ms |
| C: weather FX | weather scene, rain / snow / splash / bolt / mist shaders, ripple include (replaces stub), weather test | headless: particle ratios match the parameters, hidden underwater, flash envelope within spec and capped by reduce-flashing; storm delta ≤ 0.6 ms |
| D: seasons, surfaces, ice | surface include, rock and ice shaders, ice mask include (replaces stub), ice sheet scene, season test; edits terrain / foliage / bark shaders and vegetation | headless: ice mask CPU = shader port on 1000 points, player stays on ice over 2 m of water for 120 frames, swims under it; wet luma ≤ 0.8 x dry; surface terms ≤ 0.35 ms |
| E: living pond, fish AI | pond-life generator, plant builder, aquatic assets, life scripts / scenes / shaders, overlay include (replaces stub); edits fish scripts | generator deterministic, counts ±15%; every submerged plant below the water in its band; fleeing roach end in a bed ≥ 60%; dusk surface events ≥ 2x noon; shallows shot vs the underwater-plants reference: underwater hue 60-110°, ≥ 8% lime pixels, deep control 180-210° |
| F: audio, integration, perf, acceptance | ambience mixer, bus layout, audio assets and credits, audio test, acceptance runner, CREDITS.md merge; edits Settings, options menu, player, HUD, main scene | headless audio weights per scene, thunder delay = distance / 343 ± 0.05 s; full shot set with metrics and verdicts; perf matrix Low-Ultra x 5 views, High ≥ 60 fps; 10-minute soak clean |

## What went wrong and became rules

- A test overwrote and deleted a real scene -> tests never write `res://`.
- Tests wrote the player's settings and world state -> path overrides and persist flags on every persistent object.
- `-s` scripts with autoload names failed to compile -> `root.get_node("Name")`.
- Fauna and fish tests flaked -> `--fixed-fps 60`, seeds before deferred spawns.
- Windowed shots stole the owner's focus from a fullscreen game -> headless first, batched windowed runs with a
  warning; window-guard scripts did not help.
- Movie Maker on a GPU shared by several agents ran at about one frame per minute -> fewer, batched shots; sky-only
  shot scene for sky tuning (seconds instead of minutes).
- The overcast deck built from cumulus noise showed bright holes -> a separate slab model for closed decks.
- Exports lost data files, listed `.import` names, and dropped exported array values -> include filter,
  `ResourceLoader.list_directory`, text resources not converted, a smoke test with `--main-pack`.
- A "5 ms shadow proxy cost" was a measurement artefact (`cast_shadow = OFF` made proxies visible) -> variants hide.
- The forest cost 5-9 ms in LOD fade margins and SDFGI caused walk spikes -> margin scale per preset, 2 cascades.
- Credits from five packages -> `docs/credits_pending/<WP>.md`, merged once by the integration package.
