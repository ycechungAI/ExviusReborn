# CLAUDE.md

Guidance for Claude Code (and other AI agents) working in this repository.

## What this is

An offline, local-first Godot 4.6 rebuild of the mobile gacha RPG *Final Fantasy Brave Exvius*. Personal/educational project — no monetization, no distribution of datamined assets.

This repo is a **fork of `Hirurawa/GachaTest`** (active upstream). Before starting feature work, sync: `git fetch upstream && git merge --ff-only upstream/main` on `main`.

## Key documents

- [SPEC.md](SPEC.md) — what exists, what's missing (read this first for orientation)
- [TODO.md](TODO.md) — prioritized roadmap; update it when you land items
- [RULES.md](RULES.md) — **mandatory** architecture/coding rules
- [DATA_SCHEMA.md](DATA_SCHEMA.md) — static data and save-data shapes
- [godot/README.md](godot/README.md) — autoload list, UI navigation contract, performance rules
- [bin_parser/README.md](bin_parser/README.md) — reverse-engineered `.bin` format recipes

## Running & building

- Open `godot/project.godot` in **Godot 4.6**; main scene is `godot/demo.tscn` (F5). Fully offline — no backend or Docker.
- `godot/assets/` is **gitignored and huge**; it comes from a Google Drive dump (link in README). Without it, most scenes lack textures/data files — expect missing-resource errors in headless runs and don't "fix" them by deleting references.
- Static data: SQLite `ffbe-data.db` via the `godot-sqlite` addon (read-only, `GameDatabase` autoload) + baked JSON (`StaticDataLoader`).
- Exports: `build_pipeline.py` drives headless Godot with the presets in `godot/export_presets.cfg` (Windows, Android).
- There is currently **no test suite and no CI** — verify changes by reasoning through the service/signal flow and, where possible, running the affected scene.

## Architecture (the short version — RULES.md is binding)

- **Dumb UI**: scenes/scripts under `godot/features/` only render state and emit intent signals. No game logic, no direct state mutation, no deep `get_node("../..")` paths.
- **Services own state**: singletons in `godot/core/services/` (`UnitService`, `InventoryService`, `PartyService`, `EsperService`, `MissionService`, `QuestService`, `SwitchService`, `PlayerProfile`, …). Autoload order is defined in `godot/project.godot`.
- **Every confirmed mutation persists**: services call into `Persistence` snapshots; a feature isn't done until it survives save → load.
- **`UIManager`** is the only navigation authority: `push/pop/pop_to_root/set_root` over a menu stack; scenes are registered in its `_scenes_map`.
- **Combat** lives in `godot/features/battle/` split into `logic/` (`battle_manager.gd`, `action_processor.gd`, `target_resolver.gd`, `cover_system.gd`, `opcode_parser.gd`) and `ui/`. Battle logic communicates with UI exclusively through `BattleManager` signals.

## Conventions

- GDScript with strict static typing everywhere (`var hp: int = 0`, typed params/returns).
- Files/dirs `snake_case`; scenes/nodes `PascalCase`; signals past-tense (`item_equipped`).
- `.instantiate()` (never `.instance()`); `preload()` for hot paths; cache dynamic textures.
- Every `.gd` has a sibling `.gd.uid` file — Godot generates them; commit them, never hand-edit.
- Static data IDs are strings of digits (e.g. unit `"100000102"`); player-owned entities use UUID `instance_id`s. Don't confuse `template_id` with `instance_id` (see DATA_SCHEMA.md).

## Git workflow for agents

- Work on a feature branch; never push directly to `main`.
- Upstream remote: `https://github.com/Hirurawa/GachaTest.git`. The fork intentionally carries near-zero delta from upstream — prefer merging upstream over diverging.
- Commit messages: imperative summary line; reference the system touched (e.g. `feat(battle): apply elemental resistance in damage calc`).
