# Plan 02 — Save & Load Persistence

**Status:** Complete 2026-09-23 (Steps 1-4 + the resume prompt). All four headless test suites pass together. Only remaining gap is real on-device verification (blocked on Android export setup, a separate task).
**Goal:** Nothing currently survives closing the game, not even the permanent meta stash. Fix that, and make matching-phase progress resumable too, since mobile players expect "you left something in progress, want to restore it?"

## A distinction worth knowing before we scope this
On a phone, **backgrounding and killing are different.** Switching apps or locking the screen usually just *pauses* the game in memory, nothing lost, no save system involved, that already works for free. A save system only matters when the OS actually **terminates the process** (long time backgrounded, low memory, force-quit, restart). We can't fully control or test how aggressively a given phone does this, so the plan below assumes it can happen at any time, rather than treating it as rare.

Also worth knowing: this project has never been set up to run on an actual phone (no `export_presets.cfg` exists). Real end-to-end mobile testing of this feature isn't possible yet without doing that setup first — noted as a limitation on verification, not a blocker to building the feature.

## Three pieces, two of them now
1. **The permanent meta stash.** Small, self-contained — the thing that's actually broken right now.
2. **Matching-phase resume.** The board's state (tiles, blight, moves left, pools) is a well-defined, finite chunk of data. Doing this now, not deferring it.
3. **Battle-phase resume — deliberately *not* attempting full fidelity.** Restoring exact unit positions, HP, cooldowns, timers, and live targets is a much bigger, more error-prone undertaking (live object references, mid-flight timers, animation state all need to survive a teardown/rebuild faithfully). Instead: **if the app was killed mid-battle, relaunch restarts that same battle from its beginning**, using the board state already saved when the battle started (from piece 2). This costs almost nothing extra, since it reuses the same board-state saving, and it still delivers the "something was in progress, want it back?" moment, you'd just restart that one battle attempt rather than resume it mid-fight. Full mid-battle fidelity can be revisited later if this doesn't feel good enough once played.

## Steps

### Step 1 — `MetaBank` save/load (as originally scoped)
- `save_to_disk()` / `load_from_disk()` on `Core/MetaBank.gd`, persisting `carry_in_cap`, `permanent_upgrades`, `meta_items`, `resources` as JSON to `user://save.json`. `round_earned` stays unsaved (per-round scratch value).
- StringName dictionary keys convert to plain strings for JSON and back on load.
- A `"version": 1` field in the file, so a future format change can be detected instead of crashing on an old save.
- `load_from_disk()` runs once in `MetaBank._ready()` (autoloads init before any scene). `save_to_disk()` runs from `MetaBank`'s own `changed` signal, every mutation autosaves — the data is tiny, so this is cheap.
- **Commit 1.**

### Step 2 — A `RunState` save file for matching-phase resume
- A `run_save.json`, separate from the meta stash, holding: `Board.layout`, `tiles` (per tile: terrain type, blight, grid pos, modifiers, structure — see below), `moves_left`/`max_moves`, `round_index`, `world_blight_pressure`, `carryover_blight`, `pending_blight`, the two blight-cleared counters, `pressure_bonus`, the in-run `ResourceBank` contents, and `UpgradeRunState`'s bonuses.
- **Modifiers and structures serialize as "type id + small runtime-fields dict"**, not generically — e.g. a deposit saves its def id (rebuilds everything else from that def on load); a castle saves its current HP. Each modifier/structure class gets its own tiny `to_save_data()` / `from_save_data()`.
- Saved at the start of the matching phase, and again after each resolved move (so at worst, an in-flight swap is lost, not the whole phase).
- On game launch: if a `run_save.json` exists, offer "resume your run?" before showing the meta screen. Yes → rebuild the board from saved data instead of generating a fresh one, then re-enter matching phase (or immediately re-enter battle, per piece 3, if the save says the phase was battle).
- The save is deleted when a run legitimately ends (win or loss), so a finished run doesn't leave a stale resume prompt.
- **Commit 2.**

### Step 3 — Battle-restart-from-checkpoint (piece 3 above)
- Reuses Step 2's save: if the saved phase is `BATTLE`, resuming means rebuilding the board from the saved (pre-battle) tile data and calling the normal `start_battle_phase()` on it, effectively restarting that battle fresh rather than reconstructing its live state.
- **Commit 3.**

### Step 4 — Headless tests, same pattern as `BoardSmokeTest`
- `_Debug/MetaBankSaveTest` — set values, save, reset in-memory, load, assert round-trip; also confirm a first-ever launch (no file yet) doesn't crash.
- `_Debug/RunStateSaveTest` — build a board (reuse `BoardSmokeTest`'s setup), make a move, save, tear down, reload, assert the board matches: tile terrain/blight, modifiers, castle HP, moves left, pools.

## What I need you to check
I can verify all of this headlessly. What I can't do: watch a real phone actually get suspended and killed by the OS, or confirm the resume prompt feels right in the moment. Once export/deploy is set up (a separate task) and this is in, please test on-device: play into a run, force-close, reopen, confirm the resume prompt appears and the board looks right.

## Risks
- Step 1 is low risk, one file, no gameplay changes.
- Step 2 is the real work here: it touches how tiles/modifiers/structures represent themselves as data. I'll do it incrementally (one modifier/structure type at a time) and re-run the headless board test after each, the same discipline as the board layout work.
- Deliberately not attempting full battle-state fidelity keeps this from ballooning into a much larger, riskier effort.

## Rollback
Each step is its own commit. Worst case, delete `user://save.json` / `user://run_save.json` by hand to reset.
