# Look development loop against a reference

1. **Target.** Read the reference and its must-show list; write the numeric checks (hue bands, ratios, luma
   percentiles, projected sun position) into the design's look-target table if they are missing. Name the camera
   (`cams.json`), the world state (hour, season, weather) and the quality preset.
2. **Baseline.** One shot of today's look at that camera and state; `metrics.py` on it; a contact sheet against the
   reference. Write what misses, item by item. This is the "before" of the round.
3. **Find the owner of each miss** in the ownership table (sky shader, look table row, water export, foliage
   uniform, Environment field). A miss in a field you do not own is a note for the dispatcher.
4. **Tune cheaply.** Sky and grade on the sky-only shot scene; materials on a material preview scene; values through
   override arguments (`--look=key=value`, `--env=prop:value`, `--mat=key=value`) before editing defaults. One variable
   group at a time; keep a log (change, metric deltas).
5. **Commit values** to the table, shader defaults or exports that own them.
6. **Full-scene shots** at the target camera plus one or two neighbouring views (a different time of day, the deep
   control shot) so the fix did not break something else; a time-lapse sheet when the change depends on sun elevation
   (no pops between neighbouring hours).
7. **Accept:** metrics PASS, contact sheet looked at, one verdict line per must-show item. Keep `before_*` /
   `after_*` pairs.
8. **Cost** only if the brief carries a budget from the owner: interleaved A/B (`godot-performance`).

Windowed runs throughout: batch them (one runner script per round), one Godot window at a time, and report how many ran.

## Common misses and their usual owners

| Miss | Usually |
|---|---|
| Sky too grey at twilight | no ozone in the LUT; exposure lift too low |
| Sunset clipped to yellow | tonemapper not AgX; sun disc energy too high; glow threshold too low |
| Shadows black under trees | ambient energy / SDFGI energy low; shadow opacity 1.0 at golden hour |
| Water too blue in the shallows | no depth-based shallow pair; depth sampled at the surface point |
| Foliage flat against a low sun | no backlight boost toward the sun; translucency 0 |
| Overcast sky with bright holes | deck made from cumulus noise at high coverage; needs a slab |
| Bright scalloped cloud band at night | far cloud samples keep lit radiance; fade by distance at night |
| Plants vanish behind water | alpha blend instead of scissor / hash |
