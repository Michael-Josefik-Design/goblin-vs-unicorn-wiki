# Matching & Board review (Foundation Pass, Step 5 — 2026-09-24)

Scope: `Board/MatchingSystem.gd` (~500 lines), `Board/Board.gd` (~900), tile model in `Core/TileEntities/`. Method: read the code end to end, compare with design doc §6–7, then pin the rules down with `_Debug/MatchingRulesTest.gd` (hand-built boards + a seeded replay).

**Verdict: keep. No rewrite criterion from the plan (§5) was triggered.** The rule logic is small, readable and correct against its tests; the two real findings were minor (fixed) and one is a design decision for the developer.

## Verdict per question

| # | Question | Verdict | Evidence |
|---|---|---|---|
| 1 | Logic vs animation | **Concern (low)** | `resolve_all_cascades` awaits view animations between rule steps, so a full resolve can't run without the view. The *rules themselves* are already pure functions of `board.tiles` (`_find_match_groups`, `_compute_group_powers`, `_get_keep_for_group`, `_kill_blight_for_group`, `_compute_group_rewards`) and are tested headless without animation. Not pure-model: `Board` keeps `gems` (views) and `tiles` (model) in parallel and `collapse_columns_model` moves both. Only worth splitting if a balance simulator / AI needs thousands of headless resolves. |
| 2 | Determinism | **OK** | All randomness is Godot's global RNG (`randi/randf`, 5 sites). `seed(n)` before a run makes it reproducible: same seed + same 10 swaps gave a byte-identical board/bank; a different seed differed (`MatchingRulesTest`). |
| 3 | Edge cases | **OK + 1 design question** | Tested: straight 3/4/5 (power 1/2/3; keep-tile none/MATCH4/MATCH5), L shape (one group of 5, no keep), touching same-biome lines merge, different biomes stay separate, castle breaks a line, blight budget = match power, blight on destroyed tiles → `pending_blight`. **Resolved 2026-09-24 (developer): code was right, design doc §7.1 reworded.** (Original question: §7.1 said *all* orthogonally connected same-biome tiles join a match once a line exists; code only joins tiles that are themselves in a 3-line, so a lone same-biome tile hanging off a 3-line stays behind (test prints a NOTE). Decide which is intended.) |
| 4 | Board god object | **Concern (medium, not urgent)** | 903 lines: model ops, round/blight accounting, save/load (`get_save_data`/`load_from_save_data` ~75 lines), blight manifestation (~70), debug helpers (~50), input forwarding, phase changes. Cheap seams when next touched: a `BoardSaveIO` and a `BlightManifest` helper. |
| 5 | Performance | **OK** | Matching runs only after a swap, on ≤ 7×7 tiles. Linear `in`-array checks are irrelevant at this size; nothing runs per frame. |
| 6 | Design-doc conformance | **OK, 1 bug fixed** | Match Power = 1 + extra tiles ✔; blight destroyed = power ✔; leftover blight → pending pool ✔; waves repeat until stable ✔. **Bug found & fixed:** the per-group popup's "Chain xN" text never appeared, because the wave number was tagged onto results *after* the popup was built. `resolve_matches_async` now takes the chain index. Design doc says combo count is for "potential bonus scaling" — no scaling is applied (matches the doc). |

## Changes made in this step
- Chain index now reaches the popup (visible: "Chain x2/x3" appears on cascades).
- Removed 4 dead legacy helpers in `MatchingSystem` (`_chain_multiplier`, `_apply_multiplier_to_rewards`, `_apply_match5_keep` — which called a `Board` method that no longer exists — and a duplicate `_get_keep_pos_for_match5`). Chain *scaling* is not built; reintroduce deliberately if wanted.
- Removed a debug `print` in `Board.swap_gems` that also dereferenced a tile without a null check.
- Added `MatchingRulesTest` (18 rule checks + seeded replay).

## Recommended actions, ranked
1. ~~Decide §7.1~~ — resolved: stub tiles do not join (P&D-style); doc updated.
2. When the next feature touches `Board.gd` (blight distribution is the likely one): extract `BlightManifest` first.
3. Only if a simulator is wanted: make the resolver run on a pure model (no gems), replay animation from a log.
4. Still-unreferenced `Board.roll_terrain_type`, `add_pressure_bonus`, `get_round_state` and the tile-action popup flow (`BattleRoot._on_terrain_tile_clicked`, `WorldPopupLayer.open_tile_actions`) — review with the tile-actions design, not before.
