# Brief templates per role

Every brief is self-contained (the worker has never seen the thread) and ends with the **Waiting rule** and the **One
brief, one session** rule pasted verbatim from the dispatcher's file.

## Common header

```
Goal: <one or two sentences>.
Project: <project> (Godot <version>, <renderer>). Read first: AGENTS.md, <design doc §§>, <tools/<area>/README.md>.
Godot: <godot-exe> (headless unless this brief allows windowed runs).
Owns: <files you may create or edit>. Reads only: <files>. Never edits: <shared files>.
Contracts to keep: <node paths, groups, signals, method names, globals, data formats>.
Decisions already made: <list; do not reopen>.
Done when: <the check, with its command and expected output>.
Report: files changed (one line each), check output (decisive lines), notes for files you do not own, open items.
```

## gd-coder / gd-coder-deep

```
<header>
Change: <the feature, with the design numbers it uses and where they live (a const table, a resource)>.
Tests: add <test_x.gd> cases: <case: setup, expected value and bound>; run with --fixed-fps 60, seeded.
Minimal check only: --import, clean log, <tests> once. No windowed runs, shots or perf measures.
```

For gd-coder-deep on a bug: the owner's report verbatim, the reproduction idea, "at most 5 attempts, running log".
For a performance task the owner asked for: their words quoted, the views, the preset, the method (interleaved A/B,
variants that remove things), the budget.

## gd-look

```
<header>
Reference: <tools/ref/x.jpg> (or the checklist <file>). Must show: <items>.
Numeric checks: <metrics.py check names and bounds>.
Cameras and state: <cams.json names>, <--hour --season --weather>, quality <n>, frames <120+>.
Windowed runs allowed: yes, about <n> shots (owner cleared: "<their words>"). One window at a time, batch them.
You own: <shaders, look table rows, exports>. You do not write: <fields other owners hold>.
Report: shots, contact sheets, metrics table, verdict per must-show item, what still misses and why.
```

## gd-model-lead

```
<header>
Assets: <list with one line each: role in the game, seen from where, size, budget>.
Route preference: <generator | Blender | library>. Art direction: <references, palette, "never" list>.
Import: <loop modes, LODs, texture compression>. Contracts: <bone names, animation names, node names the code uses>.
Free first: gd-modeller, gd-modeller-go only on a rate or usage limit.
Report: per asset: paths, contract output, renders, credits rows.
```

## gd-world

```
<header>
Generator: <tools/env/gen_x.py> knobs to change: <knob: old -> new>. Determinism: two runs identical.
Placement contracts: <files other modules read and their fields>.
Checks: generator counts within <±15%> of <table>; headless world check; plan pictures.
```

## gd-audio

```
<header>
Layers and events: <list with the state that drives each>. Levels: <targets per layer>.
Licences allowed: CC0 / PD<, CC BY with attribution>. Credits file: docs/credits_pending/<X>.md.
Bus layout: <owned by this brief | read only>. Headless test: <weights per scene state, delays>.
```

## gd-video

```
<header>
Purpose: <pitch | store trailer | devlog clip>. Length: <s>. Must show: <beats>. Music: <original | CC0 track>.
Windowed recording allowed: yes, about <n> min of capture (owner cleared).
Director scripts live in: <tools/video/>. No gameplay code changes.
```

## gd-audit

```
Scope: <diff of WP-B..E | branch | whole repo>. Rules to judge against: AGENTS.md, <design doc ownership tables>.
Focus: <correctness, contracts, export risks, per-frame cost, tests>. One pass; findings ranked with path:line.
```

## gd-mechanic

```
Steps (in order, exactly):
1. <command or edit with the exact lines>
2. git add <paths>; git commit -m '<message>'
Report each command's output (decisive lines) and the commit hash. Stop and report on any failure.
```
