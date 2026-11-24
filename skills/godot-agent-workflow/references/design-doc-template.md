# Design doc template (technical, multi-package)

```
# <Feature or system>: technical design

Status: <design | approved | built (see Built blocks)>. Scope: <one paragraph>. Project: Godot <version>, <renderer>,
<target: resolution, preset, fps, hardware>.

## 0. Recommendation (read this first)
Numbered decisions, one paragraph each: the state owner, the look owner, the main technique and the rejected options
in one line each, the default start state, the package order.

## 1. Requirements and constraints
### 1.1 From the brief (fixed by the owner)
| # | Requirement |   R1 .. Rn, the owner's words where possible
### 1.2 Found in the project (the design must fit these)
| # | Constraint | Source |   C1 .. Cn: existing globals, who writes which flags today, node placement rules,
                                        bus layout, layers, test practice, "no git: disjoint files"
### 1.3 Questions only the owner can settle (defaults chosen)

## 2. Architecture
### 2.1 Overview: autoloads, main scene children, optional modules (loader), data flow per frame with
    process priorities, pause rules (who keeps running while paused)
### 2.2 State owner API (signals, read-only state, methods, the advance(dt) step), persistence (file, sections,
    validation)
### 2.3 Global shader uniforms: | Name | Type | Range / default | Writer | Readers |
### 2.4 Shared shader includes: | File | Owner | Purpose |   (stubs by the contracts package, real ones by the wave)
### 2.5 Who writes which engine field: | Object / field | Owner | Notes |
### 2.6 Contracts with existing systems: numbered patch list per system, with exact formulas and defaults

## 3..n. One section per subsystem
Options table (realism, cost, memory, rules out) and the pick; the design in tables; every number with a unit;
every new name as the code will spell it (key, uniform, node, group, file).

## n+1. Look targets
| Target | Reference | Must show | Numeric check |

## n+2. Performance
Budget table (GPU ms / CPU ms per feature, worst cases), the baseline rule, scaling per quality level.

## n+3. Test and acceptance tooling
| Tool | Purpose |   tests, shot harness and its arguments, named cameras, metrics, contact sheets, perf probe, soak

## n+4. Work packages
Order and parallelism diagram; shared-file rules; one block per package (`work-package-template.md`).

## n+5. Risks and open questions
| Risk | Impact | Mitigation |; open questions with the default used until answered.
```

Rules:

- Every row of "what goes wrong today" or "constraints" comes from code, a shot or a measured number; mark guesses.
- One source of truth per fact: name the table or module that holds each value; never "also update X".
- Start values for numbers you cannot see yet; the build's first step measures them.
- After the build, each package adds a **Built** block at the top of its section; do not rewrite the old text during a
  round.
