# Project Exvius: Reborn — Specification

This document describes what the project is, what is built today, and what the target experience looks like. For the actionable roadmap see [TODO.md](TODO.md); for data shapes see [DATA_SCHEMA.md](DATA_SCHEMA.md); for architecture rules see [RULES.md](RULES.md).

## 1. Purpose and Scope

A personal, non-commercial, educational reverse-engineering project that rebuilds the core systems of the discontinued mobile gacha RPG *Final Fantasy Brave Exvius* (FFBE) as a fully offline, local-first Godot 4.6 client.

This repository (`ycechungAI/ExviusReborn`) is a fork of the upstream project `Hirurawa/GachaTest` and tracks it closely.

### Goals
- Faithfully recreate FFBE's out-of-game loops (summon, unit growth, equipment, espers, quests) and in-game turn-based combat.
- Run fully offline: no backend, no accounts, no network calls. Local save snapshots are the source of truth.
- Keep a strict, service-driven architecture that stays maintainable as systems accumulate.

### Non-Goals
- Monetization or public distribution of datamined assets.
- Server-authoritative or multiplayer features (the original friend/companion mechanic is simulated locally).

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Client | Godot 4.6, GDScript (strict static typing) |
| Static game data | SQLite (`ffbe-data.db`) via the `godot-sqlite` addon, plus versioned JSON baked by `StaticDataLoader` |
| Player persistence | Local `user://` shadow snapshots (`Persistence` autoload) |
| Asset/data pipeline | Python tools in `bin_parser/` (map/event `.bin` → JSON blueprints), `build_pipeline.py` (headless Godot exports) |
| Platforms | Windows Desktop and Android (export presets checked in) |

Game assets (`godot/assets/`) are **not** in source control; see README for the download link.

## 3. Architecture Summary

Four principles govern all code (details in [RULES.md](RULES.md) and [godot/README.md](godot/README.md)):

1. **Dumb UI** — UI scenes are presentation-only; they emit intent signals and redraw from service state.
2. **Service autoloads** — domain state lives in singletons under `godot/core/services/` (`UnitService`, `InventoryService`, `PartyService`, `EsperService`, `MissionService`, `QuestService`, `SwitchService`, …). `UIManager` owns menu navigation with push/pop stack semantics.
3. **Local authority** — services validate and apply game rules client-side, then write a save snapshot after every confirmed mutation.
4. **Signal-first** — no deep node-path traversal; cross-feature communication goes through signals and autoload interfaces.

### Repository layout
```
godot/                  Godot 4.6 client project
  core/                 Autoload services, stat calculator, static data loaders
  features/auth/        Save-slot login / new-game / load-game UI
  features/battle/      Turn-based combat (logic/ and ui/ split)
  features/event/       Cutscene runner for parsed event scripts
  features/town/        Walkable town/dungeon exploration (tilemaps, NPCs, d-pad)
  features/outgame/     Menus: summon, units, espers, equipment, craft,
                        inventory, shop, town stores, world map, friends, profile
  features/shared/      Shared UI components and domain objects
bin_parser/             Python parsers for FFBE .bin map/event formats
build_pipeline.py       Headless export driver (Windows / Android presets)
```

## 4. Implemented Systems (current state)

### Out-game
- **Gacha/Summon** — banner summons with rate handling and summon limits; unique instance tracking per unit pull.
- **Unit management** — roster, leveling/XP, enhancement (fusion), awakening (rarity up with gil + materials), stat calculation from base arrays + equipment, unit detail views.
- **Equipment & inventory** — hybrid model: fungible stackables plus UUID-tracked unique equipment with `equipped_to` back-references; equip legality validation (slots, dual wield, categories).
- **Crafting** — recipes for equipment, abilities, and materia.
- **Espers** — ownership, star-grid enhancement board, leveling/XP, unlock gating via mission switches.
- **Party** — multiple 5-slot parties with a selected party index.
- **World map & missions** — area map, dungeon mission lists, NRG cost, mission clear tracking with objectives, first-clear rewards, `openSwitch` progression unlocks that refresh the map.
- **Shop / town stores** — store viewer with buy flow inside towns; standalone shop UI is placeholder-level.
- **Player profile** — rank, XP, NRG with timed regeneration.
- **Friends** — local friend-list state (simulated).

### Town exploration
- Tilemap towns and dungeons parsed from original `map.bin` files (two binary variants supported), NPC sprites and dialogue, camera + minimap, virtual d-pad touch controls, switch-gated locations, store and quest entry points.
- **Quests** — parsed from the database per town: delivery tasks (required items), monster-kill tasks with kill tracking from battle, reward parsing and granting, progress persistence.

### Combat (`features/battle/`)
- Turn-based flow: `INIT → PLAYER_TURN → RESOLVING_TURN → ENEMY_TURN`, multi-wave encounters.
- Actions: attack, defend, skills (opcode-parsed ability effects), item use from a 10-slot combat loadout.
- **Chain system** — frame-window-based chaining with capped damage multiplier (up to 4.0×).
- **Limit bursts** — limit gauge, limit crystal drops on kills, LB leveling.
- **Cover system** — single-target and AoE cover with mitigation states and unit-forward movement.
- **Targeting** — valid-target highlighting, dead-ally targeting (raise-type effects), simplified target resolution.
- Damage: ATK- and SPR/MAG-based formulas; dodge logic foundation is in place but incomplete.
- Enemy data loaded from the database (bestiary, encounter HP, monster parts) with safe defaults.

### Data & persistence
- Read-only SQLite (`GameDatabase`) for world/map/mission/combat/town/quest data; Android-safe local DB copy.
- `StaticDataLoader` baked-JSON cache with cold rebuild, warm load, and signature validation.
- `Persistence` snapshots after each confirmed mutation; save-slot based login (new game / load game).

## 5. Known Gaps (verified against code)

These are the largest divergences from the FFBE experience, confirmed by inspection:

- **No status effects in combat** — schema carries `status_resist` / `status_inflict`, but battle logic has no poison/blind/sleep/etc. implementation.
- **No elemental damage in combat** — `element_resist` exists in data; the action processor does not apply elements or resistances.
- **Dodge** is a foundation only (no full hit/evasion formula).
- **Espers in battle** — espers exist out-of-game; in-combat esper summons are not implemented.
- **Trust mastery** — `trust_value` is stored but has no gameplay surface.
- **Story/event content** — the event runner and parsers exist; coverage of opcodes and story flow is partial.
- **No automated tests or CI** — no test framework in the project, no `.github/` workflows.

## 6. Quality Bars

- All GDScript follows [RULES.md](RULES.md): strict typing, snake_case scripts, PascalCase scenes, past-tense signals, `.instantiate()` only.
- Every mutation flows through a service and ends in a persisted snapshot — a feature is not "done" until it survives a save/load round-trip.
- The project must import and run headless (`godot --headless`) so CI and the export pipeline stay viable.
