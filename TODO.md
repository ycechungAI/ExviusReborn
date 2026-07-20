# Development Plan / TODO

Actionable roadmap for `ycechungAI/ExviusReborn`. Context and current-state details live in [SPEC.md](SPEC.md). Items are ordered so each phase builds on the previous one; within a phase, items are roughly independent.

## Phase 0 — Fork hygiene & tooling (do first)

- [x] Sync fork with upstream `Hirurawa/GachaTest` (fast-forwarded 87 commits to `d508c01`, 2026-07-20).
- [ ] Open a PR from `claude/repo-update-plan-6w68f8` into `main` so the fork's default branch carries the sync + these docs.
- [ ] Establish a recurring upstream sync habit (weekly):
  ```bash
  git remote add upstream https://github.com/Hirurawa/GachaTest.git  # once
  git fetch upstream && git checkout main && git merge --ff-only upstream/main && git push
  ```
  Upstream is active (multiple commits/week) — sync before starting any feature branch.
- [ ] Fix `build_pipeline.py`: `GODOT_EXE` is a hardcoded empty string; read it from an env var (`GODOT_BIN`) with a sensible error message.
- [ ] Add CI (`.github/workflows/ci.yml`): headless Godot import check (`godot --headless --import`) and `gdlint`/`gdformat` (gdtoolkit) over `godot/`. This catches broken scenes/scripts without needing the asset dump.
- [ ] Add a test framework (GUT or gdUnit4) and first tests for pure logic: `StatCalculator`, chain multiplier math in `battle_manager.gd`, `EquipmentValidator`.
- [ ] Document asset-dump installation and a "runs without assets" fallback path (or a clear startup error) for contributors who clone without the Drive download.

## Phase 1 — Complete in-flight combat systems

Upstream is actively working here; keep changes small and rebase often to avoid conflicts.

- [ ] **Status effects**: implement the 8 statuses from the schema (poison, blind, sleep, silence, paralysis, confusion, disease, petrification) — infliction rolls vs `status_resist`, per-turn ticks, action gating, cure items/abilities, UI icons on unit panels.
- [ ] **Elemental damage**: apply `element_resist` / `element_inflict` in `action_processor.gd` (8 elements), show weakness/resist feedback in damage numbers.
- [ ] **Finish dodge/accuracy**: build on the "dodge logic foundation" commit — hit% formula, evade passives, miss feedback.
- [ ] **Esper summons in battle**: esper gauge, summon animation hook, esper attack damage from esper stats (out-of-game esper data already exists in `EsperService`).
- [ ] **Ability coverage audit**: sweep `opcode_parser.gd` / `skill_resolver.gd` against the datamine for unhandled effect opcodes; log-and-skip unknowns, then burn the list down.

## Phase 2 — Out-game depth

- [ ] **Trust mastery**: passive trust gain (summons/fusions), 100% trust reward item grant, display on unit detail.
- [ ] **Shop completion**: replace placeholder shop texts with a working gil shop (buy/sell, sell-price from `price_sell`).
- [ ] **Story/event flow**: expand event opcode coverage in `event_runner.gd` + `bin_parser/event_parser.py`; sequence story missions so towns, quests, and cutscenes chain into a playable early-game arc.
- [ ] **Daily/weekly structure**: NRG sinks — vortex-style material dungeons keyed off existing dungeon/mission data.
- [ ] **Save robustness**: snapshot schema versioning + migration on load; corruption fallback to previous snapshot.

## Phase 3 — Platform & polish

- [ ] Android end-to-end pass: on-device testing of DB copy, virtual d-pad, touch scroll (recent upstream fixes suggest this is fragile).
- [ ] Performance: profile texture-cache hit rates and battle frame spikes on mobile.
- [ ] Audio coverage: music/SFX hooks for battle events (chain hits, limit bursts, victory).
- [ ] Windows/Android release builds via `build_pipeline.py` in CI (artifact upload, no store distribution).

## Working agreements

- Branch per feature; PR into `main`; keep `main` fast-forwardable from upstream where possible.
- Follow [RULES.md](RULES.md) strictly — services own state, UI stays dumb, everything statically typed.
- Consider upstreaming generally useful fixes (parsers, combat correctness) to `Hirurawa/GachaTest` — smaller fork delta means cheaper syncs.
- Update this file as items land; check off with the commit/PR that closed them.
