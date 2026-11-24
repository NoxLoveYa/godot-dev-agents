# Soak test and regression runner

## Regression runner (`tools/<area>/regress.ps1 [-Only a,b] [-Retry]`)

- An ordered table `name -> (script, extra flags)`; each test runs **serially** (tests share `user://` files and the
  import cache), headless, with its flags (`--fixed-fps 60` for behaviour tests).
- Per test: log to `out/regress_<name>.txt`; count `^\s*PASS\b` and `^\s*FAIL\b` lines; count engine errors
  (`SCRIPT ERROR|Parse Error|^ERROR:` minus the exit-time leak lines); exit code; wall time; the last summary line.
- One summary line per test, then `<n> test(s) failed`; exit 1 if any failed.

```
check_world       exit 0 | PASS  14 FAIL 0 | engine errors 0 |    9 s | == summary: 14 checks, 0 failed
test_climate      exit 0 | PASS  61 FAIL 0 | engine errors 0 |   12 s | == summary: 61 checks, 0 failed
0 test(s) failed
```

- `-Retry` reruns a failed test once; use it only for a check the README names as known-flaky, and fix the flake.
- The minimal check of a brief is `-Only` the tests of the touched area; the full set runs at integration and before
  a release.

## Soak test (`soak.gd`, `soak.ps1 [-Minutes 10] [-Day 6]`)

Headless with `--fixed-fps 60` (simulated time runs faster than real time: ten simulated minutes took about two
minutes of wall time in the reference project). The main scene with every module, a short day, automatic weather and
a season cycle, for N simulated minutes:

- every simulated minute: clock, weather, season; `Performance.MEMORY_STATIC` (current and peak),
  `OBJECT_COUNT`, `OBJECT_NODE_COUNT`, `OBJECT_ORPHAN_NODE_COUNT`, `OBJECT_RESOURCE_COUNT`; mixer levels;
- a NaN / inf scan of the state owner's public state, the values it publishes as shader globals, and readable global
  shader parameters (`RenderingServer.global_shader_parameter_get_list()`);
- synthetic events on a timer (a lightning strike every 20 s, the player moved between shore, water and shallows every
  30 s) so rare paths run;
- pass criteria: no NaN; static memory, node count and resource count at the end within 12% / 5% / 5% of their value
  after minute 2 (minutes 0-2 build caches); orphans grow by at most 20; the log holds no engine `ERROR` / `WARNING`
  lines (the runner greps it, ignoring the exit-time leak lines).

A windowed soak (`-Windowed`) is the same with the renderer on; only when the owner asks.
