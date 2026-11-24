# godot-dev-agents

OpenCode / OpenChamber agents and skills for building a Godot 4 game with Claude and DeepSeek. Switch to the
`godot-dev` agent and prompt as usual: it splits each request by kind of work and hands every part to a subagent whose
model and effort are fixed in its file, so the right model does each job whatever model you would otherwise have
picked. The skills are the knowledge base the agents load: Godot architecture, headless testing, realistic rendering,
performance, the asset pipeline and the multi-agent workflow, written from the lessons of a realistic Godot 4.7
project (a forest-pond game) built with this method.

| Work | Agent | Model, effort |
|------|-------|---------------|
| Dispatch, game design, planning, architecture, contracts, briefs, checking results | `godot-dev` (primary) | Opus 5.5, high |
| Routine coding: GDScript / C# / tool scripts, features with a decided design, refactors, tests | `gd-coder` | Sonnet 5.5, high |
| Hard coding: new systems, unknown-cause bugs, networking, saves, performance, engineered shaders, code review | `gd-coder-deep` | Opus 5.5, high |
| Look development: shaders, lighting, Environment, post, water / sky / foliage, matching reference images | `gd-look` | Opus 5.5, high |
| Research: questions and option comparisons from the code, Godot docs and web (changes nothing) | `gd-research` | Opus 5.5, medium |
| Assets: plans, briefs, import settings, checks of contracts and renders | `gd-model-lead` | Opus 5.5, high |
| Assets: builds one asset in Blender or with a Python generator | `gd-modeller` | DeepSeek V4.1 Flash Free (OpenCode), high |
| The same, when the free model is rate-limited | `gd-modeller-go` | DeepSeek V4.1 Flash (OpenCode Go), high |
| World assembly: terrain, scatter / MultiMesh vegetation, placing scenes, layout checks | `gd-world` | Opus 5.5, high |
| Sound: CC0 / PD sourcing with licences, processing, buses, ambience mixers, credits | `gd-audio` | Sonnet 5.5, high |
| Trailers, showcase clips, in-game cinematics (Movie Maker + ffmpeg) | `gd-video` | Opus 5.5, high |
| One-shot review of a diff or the repo (changes nothing) | `gd-audit` | Opus 5.5, high |
| Mechanical: decided edits, re-runs, exports, docs write-back, commits, renames | `gd-mechanic` | Haiku 5.5, high |

Every subagent is prefixed `gd-`, so this pack installs next to the Roblox pack's `dev-*` agents without collisions.

## Skills

| Skill | What it holds |
|---|---|
| `godot-game` | hub: project layout, scenes / signals / autoloads / resources, GDScript style, architecture patterns (contracts, global shader uniforms, module scenes, GameFlow, Settings presets), UI and gamepad, saving, audio, Jolt physics, sharp edges, export checklist, an `AGENTS.md` skeleton for your project |
| `godot-cli-testing` | headless `-s` tests, `--fixed-fps` determinism, Movie Maker shots, contact sheets and metrics against reference images, interleaved GPU A/B, soak and regression runners, PowerShell traps |
| `godot-rendering-look` | Forward+ realism: Environment (AgX, SDFGI, fog, glow), physical-atmosphere sky with amortised volumetric clouds, water (refraction, absorption, ripples, underwater), foliage and vegetation at scale, weather and seasons through globals, the look-dev loop |
| `godot-performance` | budgets, profiling, measured lessons (LOD fade margins, SDFGI cascades, shadow gating, sky radiance), quality presets and live scaling, CPU patterns |
| `godot-asset-pipeline` | glTF import and `.import` patching, procedural generators in Python / numpy (skinned glTF), the Blender MCP route, Poly Haven API, licences and credits, Git LFS, exports |
| `godot-agent-workflow` | design doc -> contracts package -> parallel waves with file ownership -> integration -> polish -> audit; brief templates per role; a worked example |

## Install

Agents: copy the files in `agents/` to one of

- `<your project>/.opencode/agents/`: this project only, and everyone who clones it gets them.
- `~/.config/opencode/agents/`: every project on your machine.

Skills: copy the folders in `skills/` to `~/.claude/skills/` (or your project's skill folder, e.g.
`<your project>/.claude/skills/` or `.opencode/skills/`, whichever your setup reads).

Then copy `skills/godot-game/templates/AGENTS.md` to your repo root and fill it in: engine version, the path to your
Godot executable (`<godot-exe>`, ideally from an environment variable such as `GODOT_EXE`), the project folder, the
ownership table and the checks.

Reload the project in OpenChamber (or restart OpenCode), then pick `godot-dev` in the agent list.
`opencode debug agents` shows whether they loaded and which model each one resolved to.

## Models

The ids use the Claude Agent provider (`claude-agent/opus`, `claude-agent/sonnet`, `claude-agent/haiku`) and OpenCode
for DeepSeek (`opencode/deepseek-v4.1-flash-free`, `opencode-go/deepseek-v4.1-flash`). To save usage,
`gd-model-lead` sends every brief to the free `gd-modeller` first and moves it to the paid `gd-modeller-go` only when
the free model hits its rate or usage limit; the next brief tries the free one again. On another provider, change the
`model:` line in each file to that provider's id (`opencode models` lists them); the part after `#` is the effort.

## How it stays on the right models

- Each subagent's model is set in its file, not chosen at run time.
- The dispatcher may only spawn the ten `gd-*` role agents; the two modellers are reached only through
  `gd-model-lead`. Each subagent may spawn only what its role needs (mostly `gd-mechanic`), so no agent re-dispatches
  its own work. `gd-research` and
  `gd-audit` cannot edit files.
- Briefs are self-contained and results are checked before they are reported. One subagent session does one brief
  and is never resumed: a change, a fix or a re-review goes to a new session with a rebuilt brief, each sized to fit in
  200k tokens, and a running subagent finishes its brief whatever its context size.
- Briefs list the files each worker owns; within a wave no two workers own the same file.
- A model that can't be reached is reported to you; no agent silently does another role's work.
- Your project's `AGENTS.md` wins where it sets its own roles, models or rules. An explicit instruction for one task
  ("do it yourself", another model) wins for that task.

## Godot has no official MCP

The agents drive Godot through its command line: headless `-s` scripts for logic and checks, `--import`,
`--export-*`, and Movie Maker (`--write-movie`) for screenshots and video. Optional MCP servers:

- **Blender MCP** (any server that runs `bpy` in a live Blender): gives `gd-model-lead` / `gd-modeller` hands in
  Blender. Without it they use the procedural Python route.

## Windowed runs can steal focus

Every Godot run without `--headless` (Movie Maker shots, frame-time measurements, video capture) opens a focusable
window that can pull you out of a fullscreen game. External window-guard scripts and project no-focus settings did not
prevent it in practice. The agents prefer `--headless`, batch windowed runs, and the dispatcher asks you before work
that opens Godot windows. If you play on the same machine, say so in your project's `AGENTS.md`.

## Credits and inspiration

- The structure, rules and tone follow the author's [`roblox-studio-dev-agents`](https://github.com/NoxLoveYa/roblox-studio-dev-agents) pack.
- Volumetric clouds: Andrew Schneider, "The Real-time Volumetric Cloudscapes of Horizon Zero Dawn" (SIGGRAPH 2015 /
  2017, Guerrilla Games "Nubis").
- Sky and atmosphere: Sébastien Hillaire, "Physically Based Sky, Atmosphere and Cloud Rendering in Frostbite" (2016)
  and "A Scalable and Production Ready Sky and Atmosphere Rendering Technique" (EGSR 2020).
- Godot sky and cloud techniques: clayjohn's Godot demos (the amortised octahedral cloud map of
  godot-volumetric-cloud-demo-v2, MIT); only ideas are described here.
- Asset sources named in the skills: Poly Haven, ambientCG, OpenGameArt, Internet Archive, Freesound, NASA SVS.

## Licence

MIT, see `LICENSE`.
