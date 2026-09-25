# Goblin vs Unicorn — Living Roadmap

> The single place to track what's built, what's next, and what's undecided.
> Design intent lives in `Assets/GameDesignDoc.md`. This file tracks **status**.
> Update it whenever something ships, changes, or gets decided. Last updated: 2026-09-21.

**Status legend:** ✅ built (seen in code) · 🟡 partial / rough · ⬜ not started · ❓ needs verification (code exists, not yet play-tested by us)

> Note: statuses below come from a first read of the code on 2026-09-21. Nothing has been run yet, so "built" means "the code exists", not "it's bug-free". Confirming each ❓ by playing is Milestone 0.

---

## Where we are (one paragraph)

The full game loop exists in some form: **Meta screen → new run → Matching (7×7 board) → Battle (units, outposts, castle, timer) → Resolution (report + pick an upgrade) → next round**, plus a run-end and a Magic Dust stash that carries between runs. The next work is not "build the core" — it is **verify, clean up, and deepen**.

---

## Recommended build order (why this sequence)

This is the order we're building the remaining systems in, and the reasoning behind it — not a design decision, a *sequencing* one, so it can change if priorities shift. Update the ✅/⬜ as each stage actually finishes.

1. ✅ **Save/load persistence, first, before more content.** The one sequencing call worth being strict about: every system below (key items, node levels, kit unlocks, castle tiers) wants to be saved. Building persistence now means everything built after is save-ready from day one; building it later means retrofitting it into a dozen systems after the fact. Core persistence (meta stash, matching-phase resume, "resume your run?" prompt, battle restart-from-checkpoint) is done — see Phase 5 below. The "end phase early" button is also done now. Still not done: merchant shop content (the UI itself is fixed; no real merchant-only upgrades exist yet — see Phase 6/merchant row). (Deadlock detection and the mulligan were both in this bundle originally and were both dropped — see Decisions log, 2026-09-23: neither is a real state/need in this game.)
1.5 ⬜ **Economy core (decided 2026-09-25, plan `Docs/Plans/04-economy-core.md`)** — before the meta tree, because upgrades are priced against income. Steps: measure the real economy with headless simulations; change match payout to match-power; build the gatherer haul loop + gather spots; build raider walk/swing targeting; re-measure and set costs from "purchases per round" targets. The Mine/Refinery/metal-tier chain (design doc §11.3) is *vision* that belongs with item 2, not part of this.
2. ⬜ **The meta upgrade tree, generically, proven on one archetype.** Build the key-item-unlocks-a-node, gem-levels-it-up system once, correctly, then prove it on a single gatherer tree (e.g. lumberjack) before expanding to all of them. Extend the run-upgrade data model for verb-unlocks around the same time — same underlying idea.
3. ⬜ **Round out the v1 unit roster.** Build the Pikeman. Add just enough tier + ability-kit scaffolding to prove the pattern (even one ability each on Knight/Archer/Pikeman), not the full depth yet.
4. ⬜ **Leader kit for real.** Give Leader actual data: starting passive+active, unlockable slots, then the summon/field-presence and targeting system — several later ideas (the altar) depend on this existing.
5. ⬜ **Matching-phase depth.** Diagonal swap, then the special-move input system (drag/teleport/row-clear) and its slot/scroll unlocks.
6. ⬜ **Content passes**, once the systems above exist to hang them on: match-4/5 biome flavors, a devil-in-the-tile event (egg first), environmental formations, real loot tables.
7. ⬜ **World arc, mobile input, art/audio — last, deliberately**, since none of it blocks anything above.

---

## Milestone 0 — Get set up and verify (do first)

- [x] Initialise git, make first commit
- [x] Open project in Godot 4.5.1 (`/Applications/Godot.app`) and confirm it runs
- [ ] Play one full run start to finish; write down every bug/oddity in the Bug List below — *partial: played rounds 1–3 on 2026-09-24, findings logged; the battle phase is rough enough to block a meaningful full-run test, so fixing it comes first*
- [x] Decide language: stay on GDScript (see Decisions)
- [ ] Rename project from `puzzle-test` to the real game name
- [ ] Remove the unused `[dotnet]` section from `project.godot` if staying on GDScript

---

## Phase 1 — Core architecture & data models

| Item | Status | Notes |
|---|---|---|
| `TileEntity` (terrain, blight, modifiers, structure) | ✅ | `Core/TileEntities/TileEntity.gd` |
| `GridSpot` | ✅ | `Board/GridSpot/` |
| Game state machine (Matching / Battle / Resolution) | ✅ | `Core/Game.gd`, enum `GamePhase` |
| Blight state (world pool, carryover, round) | 🟡 | Lives as fields on `Board.gd`, not a dedicated Resource |
| `WorldState` resource | ⬜ | Doesn't exist yet; world/run state is scattered across `Game`, `Board`, `MetaBank` |
| Global event/signal bus | ⬜ | Systems use direct signals + `game` back-references; no central bus |
| Autoloads: `MetaBank`, `ResourceRegistry`, `StructureRegistry` | ✅ | |

## Phase 2 — Matching phase (match-3 core)

| Item | Status | Notes |
|---|---|---|
| Board size from `BoardLayout` (odd sizes, width 5–7, height 5–9), castle at centre | ✅ | Default 7×7; leaders can override; debug key **7** cycles sizes. See `Docs/Plans/01-board-layout.md` |
| Spawn, swap, gravity, cascades | ✅ | `Board.gd`, `BoardView.gd`, `BoardController.gd` |
| Group-based match detection (L/T/cross) | ✅ | `MatchingSystem.gd` |
| Wave-based cascade resolution + combo multiplier | ✅ | `_chain_multiplier`: 1.0× / 1.25× / 1.5× / 2.0× for chains 1/2/3/4+ — confirmed intentional (matches original design chat), not an arbitrary guess |
| Match-4/5 "keep" tile behaviour | 🟡 | See Phase 2 detail row above — documented in design doc §20.6 now |
| Move counter, resource rewards | ✅ | |
| Starting-board has no pre-made matches | ✅ | `_reroll_until_no_starting_matches` |
| No-valid-moves ("deadlock") detection | ⬜ (rejected, 2026-09-23) | Not needed: every adjacent swap is legal here whether or not it matches (repositioning terrain/blight on purpose is valid play), so a "stuck, no legal move" state can't occur. See design doc §22.3 correction and Decisions log. |
| "End matching phase early" action | ✅ | HUD button (`UI/main.tscn` RoundInfo row), gated to a real, always-usable `Board.end_matching_phase_early()`. Requires a confirm dialog naming the actual remaining move count before it does anything, so an accidental thumb-tap can't cost moves. Tested (`_Debug/EndPhaseEarlyTest`). §22.3 |
| Board regenerate / "mulligan" | ⬜ (cut, 2026-09-23) | Its only remaining justification ("I dislike this board") was speculative and never separately validated — see Decisions log. Not building until real playtesting shows an actual need. |
| Diagonal swap (cardinal already built) | ⬜ | 2 moves vs. 1 for cardinal; baseline for all leaders |
| Special-move system (prepped orbs, slots, scrolls) | ⬜ | Full design in `Assets/GameDesignDoc.md` §20. Consumes a regular move, doesn't add one; slots = permanent unlock, move-count-per-phase = in-run growth |
| Match-4/5 "keep" tile (modifier / chest at run center) | 🟡 | **Already built** (`apply_match4_mods_at`, `apply_match5_chest_at`) — just mountain-only for match-4, chest not biome-flavored. Closer to spec than the roadmap previously showed |
| Enemy-leader "turn" during matching | ⬜ (rejected) | Considered and explicitly rejected — enemy pressure stays passive (§20.5) for legibility/simplicity |

## Phase 3 — Blight system

| Item | Status | Notes |
|---|---|---|
| World pool + carryover tracking | ✅ | `Board.world_blight_pressure`, `carryover_blight` |
| Distribution onto grid at round start | 🟡 | Live code places blight **uniformly at random** (`Board._manifest_blight_on_tiles`). The old `BlightSystem.gd` (clustering + castle bias) was deleted 2026-09-24; its algorithm is preserved in design doc §6.2 |
| Match Power destroys blight (1 + extra tiles) | ✅ | `_kill_blight_for_group` |
| Leftover blight on matched tiles returns to pool, re-manifests same-round | ✅ | **Verified against code (2026-09-23)**: `_kill_blight_for_group` kills via Match Power budget; leftover → `pending_blight` → re-manifested onto newly-spawned tiles same round (`BoardView.gd`). Matches original design intent exactly |
| Difficulty escalation per round | 🟡 | Only `pressure_step_per_round` + a run upgrade; no clustering / biome bias rules yet |

## Phase 4 — Phase transition

| Item | Status | Notes |
|---|---|---|
| Board locks at move depletion | ✅ | `matching_phase_ended` → `start_battle_phase` |
| Gems → terrain tiles | ✅ | `Battle/Terrain/` |
| Blight → Enemy Outposts | ✅ | `Core/BlightConverter.gd` |
| Transition animation / clarity of phase change | ⬜ | Design principle: "strong distinction between phases" |

## Phase 5 — RTS battle layer

| Item | Status | Notes |
|---|---|---|
| Battle timer + pause | 🟡 | Timer works; default now 60s (exported). No speed controls |
| Unit spawning from resources / dust | ✅ | Knight, Archer, Farmer, Miner defs; Raider (enemy) |
| Outposts spawn enemies periodically | ✅ | `EnemySpawnSystem` (global round-robin tick) |
| Outpost level scales with blight | 🟡 | Design says yes; check `OutpostTileStructure` |
| Outpost clustering (merge adjacent heavy-blight tiles into one bigger outpost) | ⬜ | Full rule in §22.5 — deterministic threshold, not a chance roll. Today: strictly 1:1, `Core/BlightConverter.gd` |
| Unit brains (player / enemy / gather) | ✅ | `Battle/Units/Brains/` |
| Components (health, combat, movement, gather, armor, regen) | ✅ | Good composition-over-inheritance pattern |
| Outpost destruction purges blight permanently | ✅ | `BattleRoot._on_outpost_destroyed` |
| Structures: castle, mine, lumber mill, chest, merchant | ✅ | `Battle/Structures/` |
| Tile actions popup (tap a tile in battle) | ✅ | |
| Flying units / AIR-vs-GROUND targeting | ⬜ | Confirmed requirement in design doc |
| Collision-aware movement, range stopping, no orbiting | 🟡 | Design doc lists as near-term |
| Pikeman unit (anti-cavalry role) | ⬜ | v1 roster target: Knight, Archer, Pikeman. §23.1 |
| Unit tier axis (Knight → Paladin, key-item unlocked) | ⬜ | Uses the "Unlock, then Invest" pattern, §21.2/§23.2 |
| Unit ability-kit axis (slots + small shared pool: cleave/crit/leap) | ⬜ | Reuses the leader kit mental model, built on existing attack-cooldown system, not a new resource. §23.2-23.3 |
| Player/enemy thematic pairing (teddy bear/goblin, unicorn/skeleton horse) | ⬜ | Art direction target, §23.4 |
| More unit variety (fisherman, cavalry, mage, etc.) | ⬜ | Deferred past v1 roster |
| Enemy variety by biome + level | ⬜ | Only Raider today |

## Phase 6 — Resolution & world progression

| Item | Status | Notes |
|---|---|---|
| Resolution report overlay | ✅ | `UI/Resolution/` |
| Pick-1-of-3 run upgrade after each round | ✅ | `UpgradeSystem`, 5 upgrade defs |
| Multi-round loop with persisting blight | ✅ | |
| Victory check (world + carryover blight = 0) | ✅ | `Game._victory_check` |
| Loss → run ends, dust banked to MetaBank | ✅ | `Game.end_run` |
| Post-mortem summary (peak blight, notable modifiers…) | 🟡 | Report has some stats; design wants more |
| **Save/load persistence — meta stash** | ✅ | `MetaBank` autosaves to `user://save.json` on every change, loads on startup. Tested (`_Debug/MetaBankSaveTest`) |
| **Save/load persistence — matching-phase run resume** | ✅ | `RunStateIO` + `Board.get_save_data()`/`load_from_save_data()`. Autosaves at matching-phase start and after every fully-resolved move. Tested (`_Debug/RunStateSaveTest`) |
| **Save/load persistence — battle-phase resume** | 🟡 | Deliberately restarts that battle from its beginning rather than a live mid-fight resume (see `Docs/Plans/02-save-persistence.md`) — implemented and tested, not a gap |
| **"Resume your run?" UI prompt** | ✅ | Built entirely in code (a `ConfirmationDialog`, no scene changes) — shown on launch if a save exists. Tested (`_Debug/ResumePromptTest`) |
| Meta shop / permanent upgrades | 🟡 | `MetaShopOverlay` exists; `Core/Upgrades/Meta/` is empty |
| Magic Dust carry-in cap | 🟡 | Works; slider UI is a "later" comment |
| Castle Refinery (Refined Magic) | ⬜ | Mid/late game |

## Other systems already present

| System | Status | Notes |
|---|---|---|
| Leaders (`Leader`, `LeaderScout`: +1 move, free gatherer) | 🟡 | One leader; `LeaderAbilityDef` is an empty stub, no leader select, no field-presence/summon system, no kit-slot or cap-growth model. Full target design now in `Assets/GameDesignDoc.md` §19.4. Ability ideas (Foresight, Meteor Shower, etc.) in `Docs/history-and-vision-notes.md` |
| Castle tiers (Motte-and-Bailey → keep) + garrisoning | ⬜ | Design only, §19.2. Today the castle is a fixed structure with no tier or garrison concept |
| Unit role slots + tiered rank-up (Knight → Paladin, etc.) | ⬜ | Design only, §19.3. Today units are separate fixed defs (Knight, Archer, Farmer, Miner) with no slot/tier relationship |
| Merchant shop (mid-battle, buys run upgrades + resource trades) | ✅ | Two real bugs found and fixed: (1) dead code threw a startup error and made the feature look broken; (2) selecting "Open Merchant Shop" from the tile-action popup immediately closed the shop it had just opened — the generic popup-close-after-action step didn't know this action opened a bigger modal. Fixed via a new `TileAction.opens_own_modal` flag. `_Debug/MerchantShopTest` now exercises the real tap-popup-select path (not just calling the action directly), and was confirmed to fail without the fix. Two layout/UX fixes today: the modal panel's inner box was pinned to a hardcoded 40x40px in its scene file regardless of the code trying to make it fill the screen (fixed — now fills the board area); given a temporary red-tinted debug background + white text so it's obviously a full-screen modal, and Card icons show a checkerboard placeholder instead of hiding when there's no real icon yet. All deliberate placeholder styling, not final art. A third real bug found via screenshot: the X close button was never actually wired to anything — BoardModalPanel's close-button connection lived in an unused setup() method that nothing ever called; the only real caller used set_title() directly. Moved the wiring into _ready() so it can't be skipped. Battle art is still a placeholder sprite |
| Resource registry/bank/HUD | ✅ | Food, wood, stone, water, iron ore, Magic Dust |
| Deposit modifiers (iron deposit) | ✅ | |
| Tiered/nested loot tables | 🟡 | `Resources/GemLootTable.gd` is a flat, non-tiered stub — needs growing into the shared-subtable system, §21.1 |
| Meta upgrade tree (key-item unlock + gem-investment leveling) | ⬜ | Full design in §21.2 ("Unlock, then Invest" — now a named house pattern used across castle/leaders/moves/upgrades). `Core/Upgrades/Meta/` is still an empty folder |
| Environmental formations (moat, wall, etc.) | ⬜ | Curated diegetic patterns only — general P&D-style shape-bonus system explicitly rejected. §21.3 |
| "Devil in the Tile" special events (egg, altar, + backlog) | ⬜ | Category design in §21.4; build order is egg-style first (reuses blight's counter architecture), altar after leader summon/targeting exists |
| Player unit targeting/priority system | ⬜ | Confirmed **does not exist** — `PlayerBrain` is fully autonomous today. Needed for leader-targeted abilities (§19.4); a general rally-point/unit-command system was explicitly rejected |
| Debug tools | ✅ | `_Debug/` |

---

## Milestones (proposed order)

1. **M0 — Set up & verify** (above)
2. **M1 — Stabilise & clean up.** Fix the bug list; resolve the tech-debt list below; (battle length done).
3. **M2 — Make one round *fun*.** Tune moves / blight / battle length / unit costs; add clear feedback (phase transitions, blight numbers, outpost threat level).
4. **M3 — Battle depth.** Range/collision fixes, flying + anti-air, 2–3 enemy types, outpost levels.
5. **M4 — Run structure.** Difficulty escalation rules, win/loss polish, post-mortem screen.
6. **M5 — Meta progression.** Meta shop content, refinery, carry-in slider, more leaders.
7. **M6 — Mobile & polish.** Touch input, export to device, art/audio pass.

---

## Tech debt / cleanup list

**Done 2026-09-21**
- ✅ Leader ability was called twice per round. Harmless in effect (the first call was reset before it counted, so the Scout really gave +1), but removed the dead call and the confusing debug print.
- ✅ Dust carry-in was withdrawn twice at run start (up to 2× the cap). Removed the duplicate block.
- ✅ Two separate dust stashes in `MetaBank`: run-end dust went into `resources`, but the Meta shop spent from `magic_dust_stash`, which nothing ever filled, so upgrades could never be bought. Now one stash (`MetaBank.get_dust()`).
- ✅ Lumber mill structure id was copy-pasted as `&"mine"`; now `&"lumber_mill"`.
- ✅ Battle length default 10s → 60s (still exported; press **P** in-game to skip a phase while testing).
- ✅ Renamed `Core/BlightSystem.gd` → `BlightConverter.gd` and `DebugConroller.gd` → `DebugController.gd`.
- ✅ Removed duplicate unused `_on_outpost_spawn_unit_requested` in `BattleRoot`.

**Still open**

- No save system seen: `MetaBank` is in memory, so the stash resets when the game closes. (To confirm.)
- Big files: `Board.gd` 822 lines, `BattleRoot.gd` ~765, `UI.gd` 620. Keep extracting.
- Tight coupling: `Game` calls UI by method-name strings; `Board` and `BattleRoot` hold a `game` back-reference. Move toward signals.
- `Board._reset_round_counters()` is flagged `TODO: MOVE THIS TO ROUND RESOLUTION`.
- Blight distribution rules (clustering, castle bias, biome bias) still to be designed into the live code.
- Project is still named `puzzle-test`; unused `[dotnet]` section in `project.godot`.

## Bug list

### First real playtest — 2026-09-24 (rounds 1–3; battle phase made further testing unreliable)

Findings below came from the developer playing, then a diagnostic probe (`_Debug/BattleProbe`, not part of the regression suite) that spawned a Knight + Archer into a real battle and measured what they actually did over 25 simulated seconds.

| # | Finding | Evidence / root cause | Status |
|---|---|---|---|
| B1 | **Units get stuck and never reach their targets** (Archers freeze ~10px outside their range for 7+ seconds; Knights freeze outside melee reach; first attack took 5–8s) | Water tiles are solid physics walls (collision polygon on the water atlas tile in `main.tscn`'s TileSet) but units steer in a **straight line** at their target — no pathfinding — so any lake in the way traps them against the shore. Probe showed `v=0` with terrain speed 0.55/0.75 (not the speed multiplier), i.e. physically blocked. Enemies have the same problem (they can't reach the castle around lakes). | ✅ fixed 2026-09-24 — `BattlePathfinder` (A* on the tile grid, water solid); units follow waypoints, re-plan on a 0.35s timer (not per frame), fall back to the closest reachable point if walled off. `_Debug/BattlePathingTest`: first attacks now 0–2s (was 5–8s or never) |
| B2 | `AttackSlotManager` ("node system" for attacking structures) is effectively dead code | Nothing ever called `set_slot_manager`, so it was never wired up at all (movement recomputed its own ring point). Decision: with pathfinding + range-stop + separation, slots aren't needed | ✅ removed 2026-09-24 |
| B3 | Outposts/chests have no `get_attack_radius()` → movement and combat treat them as zero-size points (only Castle and Unit are special-cased by type) | Outpost view has `radius_px = 18` but it's never read | ⬜ open |
| B4 | Melee reach feels wrong for Knights | `UnitDef.melee_range = 50` on top of an 18px body radius ≈ attacks from ~68px, ~3.5× an outpost's visual radius | ✅ `melee_range` 50 → 14 → 4 (tune per unit in its def; melee units now touch what they hit). Units also no longer hard-collide with each other (terrain only); soft separation (radius 17 → circles touch) spaces them |
| B5 | Brains re-pick a target from scratch **every frame**, no commitment; every target change also drops the unit's attack-slot claim → target flip-flopping (probe: 9–16 target switches per unit in 25s) | `PlayerBrain.tick()` / `EnemyBrain.tick()` | ✅ fixed — new `CombatBrain` base: brains "think" every 0.4s (random phase per unit; immediately if the target dies) instead of every frame, with target commitment (a unit already being fought is kept until it leashes out at 1.5× aggro). `_Debug/BrainTest` |
| B6 | All player combat units pile onto the single enemy structure closest to the castle | `PlayerBrain` rule 2 | ✅ decided 2026-09-24: keep — focused fire on the outpost closest to the castle (defend-home-first) is the objective rule; see the priority list in design doc §10.5 |
| B7 | Battle timer is a small label in the bottom HUD, hard to see; battle is 60s and should be ~30s | `Game.battle_duration_sec = 60.0`; HUD layout | ✅ 30s; countdown bar + m:ss readout under the blight bar (`UI/BattleTimerBar`) |
| B9 | **Units still far apart in the real game** (found via the developer's screenshots after the first spacing fix) | The battle world is scaled to fit the screen (`Board.fit_world_to_rect` → `WorldRoot.scale`, ~0.5 on the dev window) but every unit distance check compared world-pixel distances against tile-pixel tunables, so spacing and reach came out ~2× too big. Headless tests ran at scale 1.0 and never saw it. Fix: `Unit.world_scale()`; reach, aggro, separation, gather range and waypoint distances all convert. `UnitSpacingTest` runs at scales 1.0 and 0.5 (verified to fail with the old behaviour: 70 tile-px apart, matching the screenshots). Also fixed: `_install_brain` left the old brain tickable until end of frame. | ✅ fixed 2026-09-24 |
| B8 | Resource costs / income numbers feel off | **Diagnosed 2026-09-25:** matching pays 1 item per tile (3× the reference game) and gatherers collect in place forever (~30 items per battle vs ~5–6 with a haul loop). Fix = economy core (build order 1.5). | 🟡 in plan |

---

## Decisions log

| Date | Decision | Why |
|---|---|---|
| 2026-09-23 | Castle: two axes — random per-run **terrain** flavor + permanent meta **tier** (unlocks verbs like garrisoning) | Keeps run variety and long-term progression separate and both meaningful |
| 2026-09-23 | Units: **role slots** (heavy melee, ranged, gatherer...) filled by **tiered rank-up** lines (Knight → Paladin); no per-unit ability-list, no gacha/shard unlocks | Real strategic choice via alternative unit lines per slot, without exploding scope or predatory unlock gates |
| 2026-09-23 | Leaders: one per run, no swap. Kit capped at 1 passive + 1 active, meta-unlockable to 2 each (verb layer); abilities grow via a **rising cap during the run** (number layer, Hades-style) | Cleanly splits permanent progression (new verbs) from run progression (bigger numbers) |
| 2026-09-23 | Leader can be **summoned onto the battlefield** (cost + aura + real death penalty); is the **one exception to indirect control** (its abilities are player-targeted, units stay autonomous) | Adds tactical agency without reversing the indirect-control pillar or requiring a full command/selection system |
| 2026-09-23 | Themed worlds (ice/fire/lava/cloud) — parked, not v1 | Real scope risk; additive later since leaders can already override board layout |
| 2026-09-21 | Stay on **GDScript**, Godot **4.5** | ~9.4k lines already written in it; faster iteration; C# gives little for this game and hurts mobile export |
| 2026-09-21 | Roadmap lives in `ROADMAP.md`, mirrored as an artifact | One source of truth in the repo; artifact is the pretty view |
| 2026-09-21 | 10-second battle is a test value → make it a setting + debug skip | Keep fast testing without shipping a stub length |
| 2026-09-21 | Board becomes dynamic in **size only: always a solid square or rectangle, no holes**, via a `BoardLayout` object | `GRID_SIZE` is hard-coded in ~11 files. Holes hurt puzzle predictability and complicate battle pathing |
| 2026-09-21 | Biome roles, cost tiers, dust-only payment, blight bonus, random castle terrain, leader passive+active, capped carry-in/out | See `Assets/GameDesignDoc.md` §17–18 |

## Biome status (code vs. design)

| Feature | Code today |
|---|---|
| Match gives biome resource + 1 dust | ✅ |
| Mine on mountain deposit (tick yield, gem chance) | ✅ (only an iron deposit def; gem loot table is a stub) |
| Farmer gathers food, miner gathers stone | ✅ |
| Lumberjack unit | ⬜ (lumber mill def exists; its script is a copy of the mine with `id = &"mine"`) |
| Farm / crops on fields | ⬜ |
| Firefly grove → dust on forests | ⬜ |
| Water use (life-support cost), fishing, boats | ⬜ |
| Terrain speed (forest .75, mountain .55, water blocked) | ✅ |
| Blight bonus on matched tiles | ⬜ (blight currently doesn't affect rewards) |
| Castle terrain random + leader override | ⬜ |
| Unit costs: tiers; dust-only payment at markup | ⬜ (units now have resource costs + a dust cost added together) |
| Carry-in for all resources; capped carry-out (win vs loss) | ⬜ (dust carry-in only; `end_run` banks everything, uncapped) |
| Chest contents | ⬜ (stub: "nothing was inside") |
| Merchant shop UI | ✅ (was already built; blocked by an unrelated startup error, now fixed) |
| Meta upgrade tree (spend gems on unit upgrades) | ⬜ (folder empty) |

| 2026-09-23 | Moves: cardinal (1) + diagonal (2) baseline for everyone; special moves are "prepped spells" that consume a regular move rather than adding one | Keeps pacing/round-length stable regardless of how many special slots a leader has |
| 2026-09-23 | Special-move slots unlock **permanently** (verb layer); total moves-per-phase can **grow during a run**, capped (number layer) | Consistent with the leader kit-slot/cap-growth split in §19.4 |
| 2026-09-23 | Special moves come from a small shared pool (Drag, Teleport, Row/Column Clear, ...); "spell scrolls" from board events teach **one specific leader** a move it doesn't know innately | Small closed option set makes an ability-pool/loadout approach safe here, unlike the combat-ability case we deferred in §19.3 |
| 2026-09-23 | Enemy leaders are passive during matching — no active "turn" | Keeps the puzzle phase legible; avoids building/tuning enemy tile-swap AI |

| 2026-09-23 | Loot tables: tiered, with **shared** sub-tables across every source (mine/chest/farm) | One place to rebalance rarity instead of duplicating it per structure |
| 2026-09-23 | Named house pattern: **"Unlock, then Invest"** — a key item unlocks a node (verb), a secondary resource levels it (number). Applies to castle, leaders, special moves, and the meta upgrade tree | Same shape kept emerging independently; formalizing it as the default template for future progression systems |
| 2026-09-23 | Meta upgrade tree: one tiered key item per node (not stacked duplicates, not a one-time global key); lower tiers convert upward (10→1), unused ones convert to a fallback currency | Spreads rarity cost across many gates instead of concentrating it on one, avoiding the "20 shards" grind feeling |
| 2026-09-23 | Environmental formations (moat, wall) are a **small curated set with diegetic effects**, not a general shape-bonus system | Avoids P&D's memorization/wiki problem; keeps matching legible |
| 2026-09-23 | "Devil in the Tile" is a category (egg, altar, + backlog), all built on one shared modifier+counter architecture (reusing blight's pattern) | Cheap to extend once the first example exists |
| 2026-09-23 | The altar is a **leader-cast ability**, not a new rally-point/unit-command system; no general player-unit-targeting system will be built | Preserves the indirect-control pillar; leader stays the sole exception |

| 2026-09-23 | No automatic board reshuffle, ever — instead one opt-in player tool: always-available "end phase early" | Board state is deliberately built here (unlike normal match-3), so an automatic reshuffle would destroy real player work; an opt-in tool gives a safety valve without the surprise cost. (Originally paired with a "mulligan" full-regenerate tool too — see next entry for why that was cut.) |
| 2026-09-23 | Cut the "mulligan" (full board regenerate) from scope entirely, not just deferred deadlock-detection | It was specced to cover two cases — "I'm stuck" and "I dislike this board." The first case turned out not to exist (swaps never require a match here, so a true stuck state is impossible). That leaves only an unvalidated variance complaint, which is also in mild tension with the reasoning that rejected automatic reshuffle (a mid-round mulligan lets a player discard their own intentional work). Not building it speculatively — if real playtesting shows bad-opening-board variance is an actual recurring problem, it's a small add then, backed by an observed need instead of a leftover justification. |
| 2026-09-23 | Outposts: 1:1 below a blight threshold; adjacent tiles above it **always** merge (deterministic) into one outpost carrying the combined total | Matches everywhere else: matching-side rules must stay predictable, not probabilistic |
| 2026-09-23 | Castle death is permanent and ends the run immediately, no in-round recovery | Castle protection is the central stake of a round |
| 2026-09-23 | Full clears are rare/optional with a large reward, but reward size and rarity are tuned together, not fixed in the abstract | Prevents an easily-repeatable full clear from deflating the world pressure curve |

| 2026-09-23 | v1 combat roster: **Knight, Archer, Pikeman only** (full target is 6: + Cavalry, Mage, Flying) | Keeps first-build scope small; Flying needs unbuilt air/ground targeting anyway |
| 2026-09-23 | Units get **tier** (rank-up) + **ability-kit** (small slots, shared pool) as two separate axes, not one big per-unit skill tree | Reuses the leader kit model exactly — one system for the player to learn, cheaper to build |
| 2026-09-23 | Unit abilities ride the existing attack-cooldown system; no new per-unit stamina/resource | Avoids an unnecessary new subsystem |

| 2026-09-23 | Round/Run/Session treated as three separate, independently-tunable clocks; run length is emergent (round length × round count), not fixed directly | Resolves "don't want it too short" and "don't want a 5-hour run" as one problem, not two |
| 2026-09-23 | Save/load persistence elevated to a near-term priority, ahead of most new content | Directly required by the pacing model (resumable sessions); also means new systems (upgrade tree, key items, kit unlocks) can be built save-ready from day one instead of retrofitted |

| 2026-09-23 | Save persistence built as three pieces: meta stash (done), matching-phase resume (done), battle resume via **restart from checkpoint** rather than live mid-fight fidelity (done, deliberate scope limit) | Live battle-state serialization (unit positions, timers, live target references) is a much larger, riskier undertaking than its value justifies right now; restarting the battle costs almost nothing extra since it reuses the same board-state save |
| 2026-09-23 | Consolidated iron-deposit-def preloads into `ModifierRules.DEPOSIT_DEFS` | One source of truth, and needed by the new save/load factory anyway |
| 2026-09-23 | Dropped "no-valid-moves" deadlock detection entirely — not building it | Traced the swap code: every adjacent swap is already unconditionally legal here, matching or not (repositioning terrain/blight on purpose is valid play per §1). A "player is stuck, no legal move" state literally cannot occur, so there's nothing to detect. "End phase early" is reframed as a plain voluntary forfeit-remaining-moves action (happy with your board, or already have what you need), not a deadlock escape hatch. |

| 2026-09-24 | Battle pathing uses **grid A\*** (`BattlePathfinder`, `AStarGrid2D` over the tile grid, water = solid) rather than Godot's NavigationRegion/NavigationAgent | The world already is a tile grid, so A* is trivial, cheap, deterministic and testable headless, and needs no TileSet surgery. Units go straight when the way is clear, follow cached waypoints otherwise, re-plan on a timer (not per frame). Flying units (future) skip it. |
| 2026-09-24 | Retired `AttackSlotManager` ("nodes around buildings") | It was never wired up (nothing called `set_slot_manager`). With pathfinding + range-stop + soft separation, standard RTS practice needs no slots at this unit count. |
| 2026-09-24 | Units collide with terrain only, not each other; crowding handled by soft separation | Hard unit-vs-unit collision jams corridors; soft push (radius 20, neighbour list refreshed ~8×/sec) spaces them without that. |
| 2026-09-24 | One shared definition of "in reach" (`Unit.melee_reach / ranged_reach / approach_reach`) used by both movement and combat | They used to compute it separately and disagreed, parking units just outside their own attack range. |
| 2026-09-24 | **All unit/battle distance tunables are in tile pixels (192 = one tile); anything compared against `global_position` distances must multiply by `Unit.world_scale()`** | The world is scaled to fit the screen, so world-pixel and tile-pixel distances differ (~2× on a phone). Defaults were re-expressed to keep the gameplay feel that developer had already approved at the ~0.48 window scale: `aggro_radius` 220 → 450, `UnitDef.ranged_range` 220 → 450 (≈ 2.3 tiles). Speeds are still world px/s (unchanged feel). |
| 2026-09-25 | **Match payout = match power (3→1, 4→2, 5→3 items per group); dust stays 1 per destroyed tile** | Matches the reference game the developer replayed; small starting numbers make each extra tile matter more. Prices are set independently of income (from "purchases per round" targets). Not built yet. |
| 2026-09-25 | **Gatherer haul loop: walk to tile, ~3s gather, carry 1, return to castle, instant drop-off; ~4 gather spots per bare tile** | Replaces stand-and-collect. Makes tile-to-castle distance matter. Capacity/time are tunable; carry capacity is a future upgrade. Not built yet. |
| 2026-09-25 | **Raider targeting: walking rule (committed > nearest visible gatherer > castle) and swinging rule (in reach: gatherer > fighter > structure)** | Castle is the goal; gatherers distract; fighters are not aggro magnets; a raider hitting the castle stays committed. Not built yet. |
| 2026-09-25 | **Vision (not committed): Mine as passive special-resource producer (chance-based tick, non-persistent); castle Refinery (ore→bars, auto-advances to next owned ore); metal tiers; bars buy meta upgrades, never per-spawn costs; castle deposit-odds selector; castle auto-builder** | Full detail in design doc §11.3. Start iron-only; belongs with the meta tree. Story: good fairy-tale creatures vs blight-powered evil ones (§2). |
| 2026-09-25 | **The dev wiki is ONE public source of truth: a GitHub Pages site + plain-text mirror in the public repo `goblin-vs-unicorn-wiki`** (supersedes the Artifact copy) | `Docs/Wiki/`: hand-written rules pages + numbers exported from the game files, so it can't drift from the code. Pages: Home, Round flow, Matching (with a playable board), Terrain, Units & combat, Structures, Economy (Income/Costs/Payback/Growth curves), Ideas & open questions. The design doc remains the source of truth for *intent*; the wiki shows behavior *as built*. Also holds vision/plan/review docs so outside reviewers (Gemini) see where the project stands. Economy pages surfaced first findings (gatherers out-earn matching on paper; dust is the tight currency) — still assumptions until measured in play. |
| 2026-09-24 | **Foundation pass (keep & repair, not rewrite); world scaling replaced by Camera2D** | Plan: `Docs/Plans/03-foundation-pass.md` (Gemini-reviewed). `WorldRoot` stays scale 1; `WorldCamera` zooms to fit the board slot, so game distances no longer depend on screen size. Supersedes the `Unit.world_scale()` patch (removed). Unit `move_speed` and projectile speeds were multiplied ~2.07× (world px/s used to be screen px/s) to keep the felt speed. Step 2 done (battle layer typed; by-name lookups confined to `Battle/BattleTargets.gd`; count via `bash _Debug/duck_typing_count.sh`). Step 3 done (dead-code sweep; report in `Docs/Reviews/dead-code-report.md`). Step 5 done (matching/board review: keep; `Docs/Reviews/matching-board-review.md`; new `MatchingRulesTest`; fixed missing "Chain xN" popup). Steps still to do: opportunistic file splits, read-only matching/board review (incl. deterministic replay test). |
| 2026-09-24 | **Match groups are P&D-style; stub tiles don't join** | A tile counts only if it is in a straight line of 3+; touching counted tiles of one biome merge (shared, stacked or offset lines all merge); a lone tile touching a line but not in its own 3+ line does not join; no diagonals. Design doc §7.1 reworded to match the code (`MatchingRulesTest` pins it). |
| 2026-09-24 | **Terrain rules for ground units: Field ×1.0, Forest ×0.75, Mountain ×0.5, Water blocked; flyers ignore terrain. Decorations are cosmetic and battle-view only; gems keep a consistent base look** | Tile decides speed/blocking, never decorations. Mountain multiplier changed 0.55 → 0.5 to match. Ridge blocking is explore-later. See design doc §8.2–8.3. |
| 2026-09-24 | Brains are efficiency-minded: think ~2.5×/sec per unit, staggered, with commitment (design doc §10.5) | Developer requirement: don't re-scan the world 25+ times per second per unit. |

## Open questions / parking lot

- Which board sizes per world/leader (rectangles allowed; odd sizes; no holes). Blight and moves need rebalancing per size.
- Chest tile behaviour (matching + battle).
- Balance: Magic Dust flexibility vs resource specificity.
- Exact blight placement algorithm & bias controls.
- Castle-in-a-match buff (castle joins a group without being destroyed → next-battle-only buff scaled by group size) — design doc §9.4; the buff depends on the castle's biome (rock→defense, forest→damage, field→gather speed/TBD); the scaling curve is a playtest question.
- Terrain decorations & neighbor-aware tiles (visual polish, cosmetic only, battle view only) — see design doc §8.3. Separate **explore-later** idea: ridge blocking between adjacent mountains (gameplay change; needs edge-level pathfinding).
- Real game name / project rename.

## Change log

- 2026-09-21: Roadmap created from code audit + design docs.
- 2026-09-21: Git initialised; first cleanup pass (see Tech debt).
