---
name: godot-agent-workflow
description: "Run a Godot project with the godot-dev agents: design doc with requirements, contracts and ownership tables, work packages with exact file ownership and acceptance checks, parallel waves, an integration package, polish, audit and acceptance; brief templates per role and a worked example from a realistic pond game. Use when planning a feature or system across agents, writing briefs, or splitting work into packages."
---

# Running a Godot project with agents

The `godot-dev` dispatcher plans and checks; role subagents build. This is the method that let a dozen agent sessions
build a day / night cycle, weather, seasons, a living pond, audio and a performance pass in one Godot project without
overwriting each other. Each rule is here because breaking it cost a pass.

## The flow

```
owner's words ─► design doc ─► contracts package (WP-A) ─► parallel wave (WP-B..E) ─► integration package (WP-F)
                    │                                              │                         │
                    └─ owner approves                              └─ each: own files,       └─ merges credits, notes,
                                                                     acceptance, report        shared files; full
                                                                                               acceptance + perf
                                         ─► polish rounds ─► audit (gd-audit) ─► fixes ─► owner hand test
```

1. **Design doc** (`references/design-doc-template.md`): recommendation first; requirements from the owner (R1..),
   constraints found in the project (C1..), architecture with process order, **contracts** (APIs, signals, groups,
   node paths, data formats, global shader uniforms with defaults and writers), the **ownership table of engine
   fields**, look targets with numeric checks, budgets, work packages, risks, open questions with defaults. The owner
   approves it (docs-only change).
2. **Contracts package first.** Declares every global uniform, stub includes, the module loader, API skeletons and
   tooling (test harness, shot harness, metrics, perf probe), so the wave's shaders and scripts compile against the
   final names. Its acceptance includes a "no-op" shot proving the stubs changed nothing.
3. **Parallel wave.** Packages with **disjoint file ownership** (`references/work-package-template.md`). Nobody edits the
   main scene, `project.godot` (except named keys), the Settings autoload, `export_presets.cfg` or `CREDITS.md`.
   Credits go to `docs/credits_pending/<WP>.md`; a change needed in another package's file is a note in the report.
4. **Integration package** (after the wave): merges credits and notes, edits the shared files (settings keys, presets,
   option rows, bus layout, removing fallbacks), runs the full acceptance set, the performance matrix and a soak.
5. **Polish rounds**: one look or feel issue per brief, before / after pairs, the owner judges pictures.
6. **Audit** (gd-audit, one pass) of the wave's diff or the repo; must-fix findings go to new coder briefs.
7. **Docs write-back** runs with each build (gd-mechanic): the design doc gets a "Built" block per package (date,
   what differs and why, measured numbers); at the end of a round, fold results into live docs and lessons into skills.

## Rules

- **One brief, one session**, never resumed; follow-ups are new sessions with rebuilt briefs carrying what the earlier
  sessions returned.
- **Resources decide the shape:** headless work runs in parallel; windowed Godot runs (shots, perf, video) are serial
  on one GPU and can steal the owner's focus: batch them and clear them with the owner first. With several agents
  rendering on one GPU, expect very slow Movie Maker frames.
- **No git in the working tree?** Then ownership must be strictly disjoint inside a wave and the dispatcher merges by
  reading reports. With git: one worktree and sub-branch per package, merged by the dispatcher (via gd-mechanic).
- **Interfaces are numbers:** where two packages meet (a depth-based colour contract between water and plants, a
  shelf height, a camera position), the design gives exact numbers and says who measures on whose built result.
- **Acceptance = headless test + pictures vs references + budget.** Each package's acceptance lists its headless
  checks with thresholds, its named shots with metric checks and a contact verdict, and its perf delta against the
  recorded baseline (perf only when the owner set a budget).
- **The owner tests by hand** at the end of each package: a numbered checklist (what to press, where to look, what to
  listen for).
- **Stop only processes you started**; start servers on your own port and stop them by PID.

## Routing

| The task | Load |
|---|---|
| Write the design doc | `references/design-doc-template.md` |
| Cut work packages | `references/work-package-template.md` |
| Write a brief for a role | `references/brief-templates.md` |
| See a full example | `references/example-pond-project.md` |
| Engine knowledge | `godot-game` and its routing |
