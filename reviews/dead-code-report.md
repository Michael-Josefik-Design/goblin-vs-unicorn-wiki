# Dead-code report (Foundation Pass, Step 3 — 2026-09-24)

Method: every `func` in non-test `.gd` files whose name appears nowhere else in any `.gd`/`.tscn`/`.tres` (60 found). Signal-by-string and `call_deferred("name")` uses would also count as a reference, so these are genuinely unreferenced.

## Deleted (superseded or duplicate; private)
- `Board/BlightSystem.gd` (whole file; algorithm preserved in design doc §6.2)
- `BattleRoot`: `_spawn_unit_at_castle`, `spawn_free_combat_unit` (thin alias), `find_modifier_by_type`
- `UI._set_spotlight_from_anchor`, `WorldPopupLayer._set_focus_ring_screen`, `BoardView._resource_name`, `Game._ui_show_resolution`

## Kept on purpose (public API for planned features)
- `MetaBank`: `deposit_dust`, `can_withdraw_dust`, `clear_all_meta_items`, `get_meta_items_copy`, `get_round_earned_copy` (meta upgrade tree / items)
- `ResourceRegistry`: `get_defs_in_channel`, `get_hud_defs`, `get_resource_names`, `is_meta_resource`, `is_run_resource`; `ResourceBank.clear_all`
- `ArmorComponent.add_flat/add_percent`, `RegenComponent.set_regen`, `MovementComponent.set_flying` (upgrades, flying units)
- `BattleRoot`: `set/add_player_damage_bonus`, `grant_temp_buff_to_player_units`, `destroy_all_*_units`, `despawn_structure` (tile actions / debug)
- `StructureView` health-UI helpers, `TerrainTile.has_open_gather_slot/setup_from_tile_entity`, `TileEntity.get_terrain_id/has_resource/take_blight_for_match`, `TileModifier.spawn_battle_props`, small `BoardLayout`/`GridUtil`/`GridSpot`/`Gem`/`Card`/`UnitBuyButton`/`BoardModalPanel` helpers.

## Deferred to Step 5 (matching review) — need a decision, not just a delete
- `MatchingSystem`: `_apply_match5_keep`, `_get_keep_pos_for_match5`, `_chain_multiplier`, `_apply_multiplier_to_rewards`. The live path uses `_get_keep_for_group`, so these look like an older implementation, **but a "chain multiplier" is a design-doc concept**: confirm whether chain scaling is intentionally absent before deleting.
- `Board.roll_terrain_type`, `add_pressure_bonus`, `get_round_state`.
- Tile-action popup flow: `BattleRoot._on_terrain_tile_clicked` (never connected) + `WorldPopupLayer.open_tile_actions/is_modal_open/set_blanket_dim` look like a half-wired feature — review with the design of tile actions.
