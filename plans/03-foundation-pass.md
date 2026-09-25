# Plan 03 — Foundation Pass

**Status:** REVIEWED by Gemini (2026-09-24), approved with small additions (§13). Steps 1, 2, 3, 5 DONE (2026-09-24); Step 4 is opportunistic.
**Audience:** (1) an outside reviewer (Gemini) who cannot see the repo — so this doc is self-contained; (2) the implementing developer-assistant (Claude) after a context reset.
**One-line goal:** Decide whether this codebase is a sound base to keep building on, and fix the specific structural weaknesses found so far — without a rewrite.

---

## 1. Why this plan exists

The developer is a first-time game developer (strong design instincts, limited coding background). The code was originally written over many back-and-forth sessions with AI chat tools, then handed to Claude Code, which acts as "head developer". After the first real playtest (2026-09-24) the battle phase was found to be badly broken in ways the code-reading audit had not caught, and the developer asked the fair question: *is the foundation itself faulty — should we rebuild from scratch, and are other systems (matching, board) built the way they should be?*

What triggered the question, concretely:

| Finding (all fixed unless noted) | What it says about the foundation |
|---|---|
| Units froze against lakes: water tiles are solid physics walls but units steered in straight lines with **no pathfinding** | Battle layer was prototype-grade |
| `AttackSlotManager` existed but **nothing ever called `set_slot_manager`** — dead, never wired | Half-finished features left over from earlier AI sessions |
| Movement and combat each computed "in attack range" **separately and disagreed**, parking units just outside their own range | Duplicated logic instead of one shared rule |
| Units looked ~2× too far apart in the real game only: the whole world is **scaled to fit the screen** (`WorldRoot.scale ≈ 0.5`), but distance code compared world-pixel distances with tile-pixel tunables. Headless tests ran at scale 1.0 and never saw it | **Architectural**: game rules silently depend on screen size |
| Earlier: merchant overlay referenced a node that was never added (startup error every launch); a modal's close button was wired inside a method nothing called; a panel was pinned to 40×40 px in its scene file | Dead / half-wired code; features "looked built" |

## 2. What the original audit actually was (honest account)

The first audit (2026-09-21) was a **read-through**: map the structure, mark things "built" if the code existed. `ROADMAP.md` says explicitly "nothing has been run yet". It flagged big files and coupling as tech debt but did **not** judge each subsystem against best practice or watch it run. The call "it's a workable base" was fair at the level of overall design and overstated what a read-through can establish. This plan is the deeper review that was missing.

## 3. Context for a reviewer who can't see the repo

- **Game:** "Goblin vs Unicorn". Round-based strategy: a **Matching phase** (match-3-style board where swaps generate resources and destroy "blight") then a **Battle phase** (the same grid becomes a battlefield; leftover blight becomes enemy outposts that spawn raiders; the player's units act autonomously — *indirect control*, no click-to-move), then a Resolution screen. Runs are rogue-like (several rounds); a meta stash persists between runs.
- **Engine/language:** Godot 4.5, GDScript (strictly typed by convention), mobile renderer, portrait 720×1280, `stretch mode = canvas_items`, `aspect = expand`.
- **Size:** ~11,200 lines of GDScript. Largest files: `Board/Board.gd` 915, `Battle/BattleRoot.gd` 768, `UI/UI.gd` 623, `Core/Game.gd` 612, `Board/MatchingSystem.gd` 517, `Battle/Units/Unit.gd` 339.
- **Scene tree (simplified):** `Main` → `Game` (phase state machine: MATCHING → BATTLE → RESOLUTION, run lifecycle, save/resume) · `WorldRoot` (Node2D) → `BoardRoot/Board` and `BattleRoot` · `UI` (CanvasLayer) · `ResourceBank`.
- **Data model:** a `TileEntity` (terrain type, blight, modifiers, structure) is the single source of truth for a tile; it is *rendered* as a `Gem` in Matching and as terrain in Battle. Model code contains no view/UI code (verified). Units are `CharacterBody2D` built from components (Health/Combat/Movement/Gather/Armor/Regen) and a swappable "brain" script; unit types, deposits and run upgrades are `Resource` files. Autoloads: `MetaBank`, `ResourceRegistry`, `StructureRegistry`.
- **Board geometry:** logical grid (default 7×7), 192 px tiles. Battle terrain is a `TileMapLayer` of 64 px cells, each logical tile = 3×3 cells; water cells have a physics polygon.
- **Existing safety net:** 9 headless test scenes (Godot `--headless`, print PASS/FAIL): board smoke, meta-bank save, run-state save, resume prompt, merchant shop, end-phase-early, battle pathing, brains, unit spacing. Commands are in `CLAUDE.md`.
- **Working rules (from `CLAUDE.md`):** small reviewable steps, one logical change per commit, plan + approval before refactors, typed GDScript, signals "call down, signal up", composition over inheritance, don't hand-edit `.tscn`/`.tres` unless necessary, never touch `.godot/`.

## 4. Evidence (measured 2026-09-24)

- Model/view separation: `TileEntity` has **no** view/UI code. `Board` already exposes `collapse_columns_model()` (a model-only version of gravity), so logic/animation separation is partly there.
- Duck-typed, string-based access in non-test code: **92** `.get("…")` lookups, **27** `has_method(…)`, **16** `.call("…")`. A typo fails *silently*; this is precisely how the "slot manager" and radius bugs hid.
- **21** `game.` back-references (`Board`, `BattleRoot` hold a `game` pointer; `Game` calls UI by method-name strings).
- Four files > 600 lines (see §3).
- **No `Camera2D` anywhere**; the world is fit to the screen by scaling a Node2D (`Board.fit_world_to_rect` sets `WorldRoot.scale`).
- Screen↔world conversion is scattered over ~10 call sites (listed in Step 1).
- Dead code: `Board/BlightSystem.gd` (self-described "not called by anything", on an obsolete data model). A systematic unreferenced-script scan found nothing else, but dead *methods* inside live files are not covered by that scan.
- **Possible latent bug to verify:** `UI._update_board_slot_layout()` calls `battle_root.fit_world_to_rect(r)` during the battle phase, but the only `fit_world_to_rect` defined in the repo is on `Board`. If confirmed, any window-resize during battle raises an error. (Handled inside Step 1.)

## 5. Decision: keep and repair; do not rewrite

**Recommendation: strangler-style repair — replace weak subsystems one at a time behind tests.**

Reasons:
1. The developer's real asset is the *design* (`Assets/GameDesignDoc.md`, ~700 lines, now recorded) — it survives any code decision.
2. The architecture at the level that matters is sound: data-driven defs, component composition, model/view split, small autoloads.
3. Substantial subsystems have already been replaced in place without cascading damage (board layout constants → `BoardLayout`; persistence; pathfinding; brains). That is evidence the code is modular, not tangled.
4. A rewrite discards ~11k working lines, and the same classes of mistakes recur unless the same reviews are done anyway.
5. The test net (9 suites) now makes refactoring safer than at any earlier point.

**Criteria that would flip this to "rewrite":** Step 5's review finds that matching/board logic cannot be tested or changed without touching most of the codebase; or Step 1 shows the Camera2D change requires reworking most input/UI code (i.e. coupling to screen scale is pervasive, not ~10 sites); or repeated fixes keep exposing the same *unrelated* structural defect in different subsystems.

## 6. Principles for the pass

- Every step is **independently shippable and independently revertible** (one commit or a short commit series; `git revert` is the rollback).
- **No behavior change** except where a step says so. Where behavior *does* change (e.g. removing the scale patch), the change must be expressed as a test first.
- All 9 suites green before and after every step. Rare test flakes are **diagnosed, not ignored** (this session's flakes each turned out to be a real cause: a stray timer, a stale brain, a scale flip).
- Tests must run under conditions the real game uses (the scale bug proved headless-at-scale-1.0 is not enough): battle tests run at ≥2 world scales/zooms until Step 1 removes the variable.
- Docs updated in the same step: `ROADMAP.md` (status + decisions log), `CLAUDE.md` if conventions change.
- Order is by "removes a whole class of bug" value, cheapest first where equal.

---

## 7. Steps

### Step 1 — Replace "scale the world" with a Camera2D

**Problem.** `Board.fit_world_to_rect` sets `WorldRoot.scale` to fit the screen (~0.5 on the dev window). Everything under `WorldRoot` (board, battle map, units, physics) is therefore in a scaled coordinate space. Game rules (reach, aggro, separation, gather range) had to be converted with a `Unit.world_scale()` patch. A rule that depends on screen size is a design bug; scaling a Node2D root is a well-known Godot footgun (global-space distances, physics, and any code using `global_position` all inherit the scale).

**Target design.** `WorldRoot` stays at `scale = 1`. One `Camera2D` (zoom = fit factor; positioned so the board is centred in the UI's "board slot" rect) provides the view fit. `CanvasLayer` UI is unaffected by cameras. `get_global_transform_with_canvas()` already includes the canvas (camera) transform, so world→screen conversions built on it keep working unchanged.

**Files/sites to change (audit found these):**
- `Board/Board.gd:268 fit_world_to_rect` — replace scale/position with camera zoom/offset; keep the same rect-based API so `UI._update_board_slot_layout` callers don't change.
- `Board/BoardController.gd:161 _screen_to_grid` — uses `board.to_local(screen_pos)`, which assumed a scale-only transform. Must go screen → canvas/world (via the canvas transform) → local.
- `Battle/BattleRoot.gd:191` — `to_local(get_viewport().get_mouse_position())`, same issue.
- `Battle/BattleRoot.gd:394 tile_to_screen_pos`, `:747 get_board_screen_rect`, `Board/Board.gd:292 get_board_screen_rect`, `UI/UI.gd:437`, `World/WorldPopupLayer.gd:157` — world→screen; expected to keep working; **verify, don't assume**.
- `UI/UI.gd:600 _update_board_slot_layout` — contains the possibly-nonexistent `battle_root.fit_world_to_rect` call (see §4).
- **Remove** the `Unit.world_scale()` patch and every `* world_scale()` / `aggro_distance()` conversion added on 2026-09-24 (Unit reach fns, `MovementComponent` separation/waypoint/stop/neighbour distances, `GatherComponent.is_in_range`, brains' aggro). Keep the *convention* (tunables are tile pixels) documented; it becomes true by construction.
- Keep `aggro_radius`/`ranged_range` at 450 tile-px (already re-expressed to preserve approved feel).
- Unit speeds are currently world px/s and were tuned at scale ≈0.48 (i.e. units effectively move at 240 *screen* px/s). After the change, world px/s = tile px/s; to preserve the felt speed, speeds should be multiplied by ~1/0.48 ≈ 2.07 (or the value re-tuned). **Flag: this is a deliberate behavior-preserving retune, verify by eye.**

**Acceptance criteria.**
1. `WorldRoot.scale` is `(1,1)` at all times in both phases.
2. `UnitSpacingTest` passes with the *camera* zoom set to 1.0 and 0.5 and with no `world_scale()` code present (test rewritten to drive camera zoom).
3. New input-mapping test: for both phases and ≥2 zoom values, a synthetic click at the screen position of a known tile centre resolves to that tile (matching swap input and battle tile-tap).
4. New test: `get_board_screen_rect()` (both phases) equals the expected on-screen rect under zoom.
5. Manual: developer confirms visual layout unchanged (board centred in slot, popups/floating text/overlays aligned, crisp pixel alignment — camera position rounded).
6. Window-resize during battle raises no error (the latent-bug check).

**Open technical questions for the reviewer:** Camera2D vs a `SubViewportContainer` with fixed-size world? (Camera chosen because it's the standard lighter option and keeps physics/input simple; SubViewport would isolate scale completely but complicates input forwarding and UI overlay alignment.) Any interaction with `stretch mode = canvas_items` / `expand` that would make a camera zoom double-apply? Best practice for keeping pixel art crisp with fractional zoom (`Camera2D` position rounding, texture filter)?

**Risk:** medium (touches input and overlay alignment). **Rollback:** revert the series. **Size:** M.

---

### Step 2 — Type the battle layer; confine duck-typing to one place

**Problem.** Battle code reaches into other objects by string (`_unit.get("_target_node")`, `_unit.get("def")`, `b.get("gather_terrain_type")`, `has_method("take_damage")`, `.call("take_damage", …)`). Typos fail silently; refactors can't be checked by the parser; private-by-convention fields (`_target_node`, `_target_pos`) are read from other classes.

**Approach.**
- Give `Unit` a proper typed, public surface (`target_node`, `target_pos`, etc.) instead of underscore fields read via `get()`; components already have typed `_unit: Unit` after this session's work, so most string lookups become direct property access.
- Targets are heterogeneous (Unit vs. structure *views* — Castle, Outpost, Chest — which are plain `Node2D`s and can't share a base with `CharacterBody2D`). Put the unavoidable duck-typing behind **one** small static helper (e.g. `BattleTargets.is_alive(n)`, `.radius_of(n)`, `.apply_damage(n, amount)`), so string-based access exists in exactly one file. Alternative for the reviewer to weigh: a tiny `Damageable` component node attached to anything targetable, giving one typed lookup path.
- `Unit._configure_brain_from_def` (`b.get("gather_terrain_type")`) → typed check for `GatherBrain`.
- Baseline to beat: 92 / 27 / 16 (`.get("…")` / `has_method` / `.call("…")`). Goal: battle layer near zero outside the helper. Track the numbers in the commit message.

**Acceptance.** All 9 suites pass; a grep-based check (script in `_Debug/` or documented command) reports the new counts; no behavior change.
**Risk:** low–medium (mechanical, parser-checked). **Size:** M.

---

### Step 3 — Dead-code sweep

**Approach.** (a) `Board/BlightSystem.gd`: its design ideas (clustering, castle-proximity bias, "prefer existing" weighting) are still wanted for the roadmap's blight-distribution work — move them into the design doc (§6) first, then delete the file. (b) Detect dead *methods and unused exports* inside live files (script: list `func` names never referenced elsewhere; review by hand — Godot signal connections and `call_deferred("name")` strings mean naive detection produces false positives). (c) Confirm/repair the latent `battle_root.fit_world_to_rect` call if Step 1 hasn't already removed it. (d) Delete only what is proven unused; one commit per deleted cluster.

**Acceptance.** All suites pass; deleted items listed in the commit; nothing referenced by any `.tscn`/`.tres`. **Risk:** low. **Size:** S.

---

### Step 4 — Split the large files along existing seams (opportunistic, not big-bang)

**Rule.** Only split when we're already changing that area for a feature or fix, and only along a seam that already exists. Never a pure "reorganise" commit without a reason.

**Candidate seams (to be confirmed by reading, not assumed):**
- `BattleRoot.gd` (768): unit spawning/tracking, board-tap input, structure tracking, tile-modifier ticking, end-of-battle accounting, the debug helpers.
- `Board.gd` (915): model operations (gravity/collapse already has a `_model` variant), matching orchestration, round/blight accounting, input, save/load (`get_save_data`), phase transitions. Possible extraction: a `BoardModel`-style object with no `Node` dependency (also helps testing and future AI/simulation).
- `Game.gd` (612): run lifecycle vs. save/resume vs. dialog building (`_offer_resume_prompt`, `_on_end_phase_early_requested` build dialogs in code) — extract a run-save manager and a dialogs helper.
- `UI.gd` (623): unit buy buttons, meta overlays, board modals.
- Also reduce `game.` back-references by turning method-name-string calls (`ui.call("show_meta")`) into signals or typed calls as each file is touched.

**Acceptance per split:** suites green; public behavior identical; file sizes recorded. **Risk:** medium per split. **Size:** M each (spread over time).

---

### Step 5 — Read-only review of the matching system and board

**Purpose.** Answer the developer's question ("is matching/board set up the way it should be?") with evidence, not assumption. **No code changes in this step unless a real bug is found.** Deliverable: `Docs/Reviews/matching-board-review.md` plus any new tests.

**Questions to answer:**
1. **Logic vs. animation:** `MatchingSystem` interleaves rule resolution with `await`ed animations (6 `await`s). Can the resolution be run on the model only (fast, deterministic, testable, simulatable) with animation replayed from a result log? `collapse_columns_model()` suggests the pattern already exists partially.
2. **Determinism/testability:** is randomness centralised and seedable? Can a test build a specific board and assert exact outcomes (match groups, keep-tile rules for 4/5-matches, blight destruction budget, chain multiplier, leftover-blight re-manifest)?
3. **Edge cases:** cascades with modifiers/structures on tiles; blight moving with tiles through swaps; castle tile handling; odd board sizes (5–7 × 5–9); match-4/5 keep-tile behavior; swaps that produce no match (allowed by design).
4. **`Board` responsibilities:** is it a god object (model + orchestration + input + save + phase)? Which seams (Step 4) are cheap?
5. **Performance:** any per-frame scans or allocations in matching/board views.
6. **Design-doc conformance:** rules in `GameDesignDoc.md` §7 (group-based matching, Match Power, wave resolution) vs. code.

**Output format:** a verdict per question (OK / concern / bug), each with file+line evidence, and a recommended action list ranked by value. Feeds the rewrite-or-keep decision (§5 criteria).
**Risk:** none (read-only). **Size:** S–M.

---

## 8. Sequencing & dependencies

1. **Step 1 first** — largest class-of-bug removal, and every later battle test is simpler once scale is not a variable.
2. **Step 5 can run in parallel** (read-only) and should finish before any matching/board refactor.
3. **Step 2** after Step 1 (both touch Unit/Movement/Combat; do the coordinate change first, then the typing).
4. **Step 3** after Step 2 (fewer false "dead" candidates once lookups are typed).
5. **Step 4** continuous, tied to feature work.
6. After Steps 1–3 + Step 5's verdict: resume the roadmap (resource/cost tuning pass B8, merchant content, meta upgrade tree — see `ROADMAP.md` "Recommended build order").

## 9. Out of scope

Art/styling changes; new gameplay features; Android export setup; save-format changes; rebalancing (except the deliberate speed-preserving retune in Step 1); rewriting the design doc.

## 10. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Camera change subtly misaligns popups/overlays/floating text | Explicit alignment tests + developer visual check before merging; keep old code path revertible as one commit series |
| Tests give false confidence (as the scale bug showed) | Run battle tests at multiple zooms; add input-mapping tests; treat any flake as a bug to explain |
| Refactor churn stalls feature work | Steps 1–3 are bounded; Step 4 is opportunistic; Step 5 is read-only |
| Behavior drift from typing/refactors | "No behavior change" rule + full suite before/after; unit speeds retuned only in Step 1, deliberately |
| Scope creep into a rewrite | §5 flip criteria are explicit; revisit only if a criterion is met |

## 11. Questions for the reviewer (please challenge)

1. Is "keep and repair" the right call given ~11k lines, a first-time developer, and the evidence in §1/§4? What would you check that we haven't?
2. Step 1: Camera2D vs. SubViewport vs. keeping the scaled root but fixing all distance code once — which would you pick for a portrait mobile game with a fixed logical board and a `CanvasLayer` UI, and why? Any pitfalls with `canvas_items`/`expand` stretch?
3. Step 2: single duck-typing helper vs. a `Damageable` component vs. a shared base class (given `CharacterBody2D` units and `Node2D` structure views)?
4. Is the ordering right? Anything that should come *before* Step 1 (e.g. a deterministic RNG seed, a headless "screenshot"/layout assertion harness)?
5. Are there other common Godot 4 architecture smells a codebase like this (autoloads + signals + `game` back-references + code-built dialogs) typically has that §4 didn't measure?
6. For Step 5, what additional tests would you want around a match-3 resolver before trusting it as a foundation?

## 12. Decision-log entries to add after approval (do not add yet)

- Keep-and-repair over rewrite (with the §5 flip criteria).
- World scaling replaced by Camera2D (or the reviewer-chosen alternative); tile-pixel convention stands.
- Duck-typing confined to one helper in the battle layer.
- Foundation pass ordering.

---

## 13. Review outcome (Gemini, 2026-09-24)

Verdicts: (1) keep-and-repair confirmed; (2) Camera2D chosen over SubViewport/scaled root; (3) single static helper (`BattleTargets`) chosen over Damageable component/base class; (4) ordering confirmed; (5)/(6) additions below.

Additions folded into the steps:
- **Step 1:** verify project default texture filter and round the camera position so pixel alignment stays crisp. (Project uses `canvas_items` + `expand`; check zoom isn't double-applied.)
- **Step 2:** while typing, also audit hard-coded `get_node("../..")` paths across scene boundaries (prefer `@export`/injection) and `queue_free()`-then-use races (null/`is_instance_valid` checks).
- **Step 5:** add a headless **deterministic replay test**: seed the RNG, feed a fixed sequence of ~10 swaps, assert exact grid state, resource yields and blight removal counts, and that two runs are identical. Step 5 must also confirm the model resolves swaps with no view dependency.
