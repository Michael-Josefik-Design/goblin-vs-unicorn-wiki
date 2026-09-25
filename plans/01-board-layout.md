# Plan 01 — Dynamic Board Size (BoardLayout)

**Status:** Approved 2026-09-21 — Steps 1–5 implemented; waiting on your visual check of non-7×7 boards. Written 2026-09-21. Revised after review (odd-size validation in Step 1, playtest checkpoints in Step 2, safety test moved before rectangles).
**Goal:** The board can be any solid rectangle (e.g. 5×5, 5×7, 7×7, 7×9), chosen per world/leader. No holes or odd shapes (decision recorded in the design doc).

## Mental model
Today the board is a sheet of graph paper that is *always* 7×7, and about a dozen files each carry their own copy of "7" (as `GRID_SIZE`, used ~49 times). We replace that with **one `BoardLayout` object** that everyone asks: "how wide are you, how tall, where's the centre, is this cell on the board?" Then a different size is just a different `BoardLayout`.

## What does NOT change
- Tile size stays **192px**. The terrain art (3×3 blocks of 64px tiles) depends on it. A smaller board is simply scaled up on screen by the existing `fit_world_to_rect`, so tiles look bigger. No art changes.
- Matching rules, blight rules, gravity, and battle rules stay exactly as they are.

## Step 1 — Introduce `BoardLayout` (no visible change)
- New `Board/BoardLayout.gd` (a Godot `Resource`): `width`, `height`, `tile_size`, plus helpers: `pixel_size()`, `center()`, `in_bounds(pos)`, `neighbors_4(pos)`.
- `BoardLayout` **validates itself**: width and height must be odd and within limits (min 5, max 7 wide / 9 tall). A bad value prints a warning and falls back to the nearest valid size, so a bad leader config can't break the castle-centre math.
- `Board` gets `@export var layout: BoardLayout`, defaulting to 7×7. Set per world/leader later.
- **Commit 1:** the new class only. Game runs identically.

## Step 2 — Migrate every file off `GRID_SIZE`
Replace each `GRID_SIZE` use with `layout.width` (for x loops) or `layout.height` (for y loops). Done one file at a time, in this order. I run a headless load check after each file; **you play the game at three checkpoints (after group 2, after group 4, and at the end)**, not after every file:
1. `Board/Board.gd` (~24 spots: grid building, gravity, spawning, castle centre, layout/fit)
2. `Board/BoardView.gd`, `Board/BoardController.gd`, `Board/MatchingSystem.gd`
3. `Board/GridUtil.gd` and `Board/BlightSystem.gd` (currently take one number; will take the layout)
4. Battle side: `BattleRoot.gd`, `StructureSpawner.gd`, `TerrainTileRenderer.gd`, `GatherBrain.gd`
5. `Core/MerchantSystem.gd`, `Core/BlightConverter.gd`, `_Debug/BoardDebugOverlay.gd`
- Finish by **deleting** the `GRID_SIZE` / `BOARD_PIXEL_SIZE` constants. Godot then flags any spot I missed the moment the project loads. That's our safety net.
- Careful: while the board is still 7×7, mixing up width and height is invisible. That's why the safety test (Step 3) comes before rectangles.
- **Commit 2:** "Board reads size from BoardLayout". Still 7×7, still identical to play.

## Step 3 — Safety test (before rectangles)
A small headless check, written *before* rectangles work so it starts red and turns green as we fix things (`_Debug/BoardLayoutTest`). It builds boards of 5×5, 5×7, 7×7, 7×9 and asserts: no missing tiles, castle inside the board and not on water, no starting matches, gravity leaves no gaps. It builds a 5×7 board from the first run, so a swapped width/height fails immediately. Run from the terminal in seconds, repeatable after any future board change. I can't watch the game visually, so this plus your playtests is how we verify.
- **Commit 3:** "Add board layout safety test" (5×7 case expected to fail until Step 4).

## Step 4 — Make rectangles actually work
Places that quietly assume *square*:
- `MatchingSystem._find_raw_matches` scans both directions with one length.
- `Board.fit_world_to_rect` and the board background use one side length for width and height.
- `BattleRoot` input area and screen-rect code use one side.
- Castle placement: `get_center` returns `(size/2, size/2)`.
- **Commit 4:** "Support rectangular boards".

## Step 5 — Choose the size per world / leader
- `Leader` gets an optional `board_layout`; if set, `Game` gives it to the board at run start. Otherwise the default 7×7.
- Size is fixed for a whole run (the board still rebuilds every round, at the same size).
- Guard rails already live in `BoardLayout` (Step 1).
- **Commit 5:** "Leaders can set board size". Includes a second test leader (or debug key) on 5×5 so you can see it.

## Risks and things to decide
1. **Even sizes and the castle.** With an even width (e.g. 6) there's no true centre cell; the castle would sit off-centre. **Proposal: allow odd dimensions only (5, 7, 9) so the castle is always exactly central.** Even sizes can come later if you want them.
2. **Phone tile size (my estimate, not measured).** The 7-wide board on the 720px-wide viewport is about 90px per tile, roughly 50pt on a typical phone, near the comfortable-touch limit. Rough guide: **width ≤ 7, height ≤ 9.** 9-wide boards would be too small to tap comfortably in portrait. Taller-than-wide boards fit well because the screen is portrait.
3. **Balance shifts with size.** Blight and moves are tuned for 49 cells. A 5×5 (25 cells) with the same blight is much denser. Not solved by this plan; we tune later (blight per cell is the likely fix).
4. **Mid-run size changes** would touch persistent state, so size is fixed per run.

## Outcome notes (2026-09-21)
- Step 2 migrated width/height correctly as it went, so the Step 3 test passed for rectangles right away; Step 4 needed no separate fixes.
- The debug key **7** cycles 7×7 → 5×5 → 5×7 → 7×9 and restarts the run, for visual checks.
- `Board/BlightSystem.gd` turned out to be unused; moved to `BoardLayout` but left dormant.

## Definition of done
- Game plays identically at 7×7 (regression check by you).
- A 5×5 and a 5×7 board play through matching → battle → resolution without errors.
- `GRID_SIZE` no longer exists in the code.
- Roadmap and design doc updated.

## Rollback
Every step is its own git commit, so any step can be undone on its own.
