# Goblin vs Unicorn — Living Design Document (Updated Handoff)

> **Purpose:** Single-source “handoff” doc for new collaborators (including AI tools).  
> **Status:** Living design document — update as systems/features evolve.  
> **Engine/Stack:** Godot 4.x (GDScript), mobile-first portrait layout.

---

## 1. High-Level Vision

**Goblin vs Unicorn** is a **round-based strategy game** blending:

- **Enhanced match-3** (resource generation + board manipulation + pressure reduction)
- **Lightweight RTS battle phase** (indirect control, real-time timer, units + structures)

Each round has two phases:

1. **Matching Phase**  
   Player swaps gems to gain resources, reposition threats, and partially reduce Blight.
2. **Battle Phase**  
   The same grid becomes a battlefield: gems become terrain; unresolved Blight becomes enemy structures that spawn units.

Progression is **world/run-based** (rogue-like per world, not per round). Enemy pressure persists between rounds until fully purged.

---

## 2. Core Fantasy & Theme

- Classic **Good vs Evil** framing in a whimsical fantasy world.
- Player represents the “good” faction defending a magical Castle.
- Enemy threat is the spreading **Blight**, corrupting land and spawning **Outposts** and enemy units.
- **World & story (developer, 2026-09-25):** a Shrek-style mash-up fairy-tale world where every story creature lives together — the good side (Santa, leprechauns, the Tooth Fairy…) and the evil side (witches, mummies, Dracula…). The **Blight is pure evil**, powering the evil creatures and trying to take over the good side's world. The leaders the player uses are the good side's champions. Upgrading the castle can change the world around it (see §11.3 deposit odds).
- Worlds/leaders change “rules of nature” (terrain overrides, Blight distribution rules, special match rules).
- Visuals: watercolor, hand-drawn, soft/pastel palette with paper textures; clarity > realism.

---

## 3. Platform & Technical Constraints

- **Engine:** Godot 4.x
- **Target:** Mobile-first (portrait), optional desktop later
- **Design priority:** Clear feedback loops and legibility; scalable architecture; avoid scope creep early.
- **Current technical direction (battle):**
  - Board generates terrain visuals from tile data.
  - Units currently “move toward a target” with basic collision/walkability checks.
  - Systems are being extracted from a bloated `BattleRoot` into focused nodes (Spawner, Merchant, Blight conversion).

---

## 4. Core Game Loop (Round)

### 4.1 Round Start
- A fresh grid is generated.
- Castle tile placed at center.
- A **Round Blight Pool** is calculated from unresolved world pressure.
- A portion of pool is manifested onto the board as **Blighted Tiles**, using placement rules.
- Remaining blight stays in the pool and may re-enter later rounds.

### 4.2 Matching Phase
- Player has a finite number of moves.
- Tiles (gems) represent biomes/resources and can carry modifiers.
- Matching 3+ tiles:
  - Grants resources
  - Removes tiles from board
  - **Destroys Blight** according to **Match Power**
- Tiles fall via gravity; modifiers move with tiles.
- **Blight moves with tiles** (see “Blight Ownership Model” below).

### 4.3 Transition to Battle
- Board locks.
- Unresolved Blight converts into **Enemy Outposts**.
- Tiles convert into corresponding **Terrain Tiles** (battle map).
- Round Blight Pool is frozen for this battle snapshot.

### 4.4 Battle Phase
- Player spawns units using earned resources.
- Enemy Outposts spawn enemies on an interval.
- Enemy units advance toward Castle.
- Player has indirect control: no manual pathing; brains decide behavior.

### 4.5 Battle Resolution
- Blight destroyed in combat becomes permanently removed from world pressure.
- Undestroyed Outposts return their contained blight to World Pool.
- Battle stats recorded (outposts destroyed, units killed/lost, castle damage, etc).

### 4.6 Round End
- Board is cleared/reset.
- World pressure persists.
- Next round begins.

---

## 5. Board, Grid & Tile System

### 5.1 Grid Size & Layout
- Grid size: **7×7**
- Cell size: **192px**
- Board is composed of **persistent Grid Spots** (fixed positions; do not move).

### 5.2 Grid Spots (Anchors)
Grid Spots:
- Do not move
- Do not reset during cascades
- Do not store Blight
- Act as anchors for tile positioning and visuals

### 5.3 Tiles (Unified Entity)
A **TileEntity** occupies a Grid Spot and persists across phases.

Each tile has:
- **Biome Type:** Field, Forest, Mountain, Water, Castle
- **Blight:** integer
- **Modifiers:** array (bonuses/special rules)
- Render form:
  - **Gem** during Matching
  - **Terrain Tile** during Battle

**Important:** Blight belongs to the **tile**, not the grid spot.

---

## 6. Blight System (Design + Implementation Direction)

### 6.1 What Blight Represents
Blight is unresolved enemy pressure:
- Applied at Round Start
- Moves with the tile through swaps and cascades
- Converts into enemy structures if unresolved at battle transition
- Removed only by:
  1) **Matching** (partial destruction)
  2) **Battle** (full destruction when outpost dies)

### 6.2 Blight Placement Rules (Round Start)
Default constraints:
- Cannot spawn on Water (by default)
- Cannot spawn on Castle (unless world rules allow)

World/leader rules can override:
- clustering bias
- castle proximity bias
- biome bias
- special rules like “blight spreads on cascades”

**Prior algorithm sketch (from the retired `BlightSystem.gd`, 2026-09-24; not implemented — live code places blight uniformly at random in `Board._manifest_blight_on_tiles`):** each new blight point is placed one at a time from two pools, *already-blighted* and *clean* tiles.
1. `prefer_existing_weight` (0..1, was 0.7): chance to draw from the already-blighted pool, so blight thickens where it already is.
2. `cluster_chance` (0..1, was 0.45): chance to instead pick a random existing blight tile and place on a 4-neighbour of it (if that neighbour is in the chosen pool) — grows patches.
3. `castle_bias_strength` (0..1, was 0.6): weighted pick where each candidate's weight is `1 + bias × (max_dist − manhattan_dist_to_castle)`, floored at 0.01, so blight leans toward the castle.
4. `allow_blight_on_castle_start`: excludes the castle tile from the pools.
All four are per-world/leader tunables. Rewrite against `TileEntity` (not grid spots) when the distribution work is scheduled.

### 6.3 Blight Interaction During Matching (Core Rules)
When a match group resolves:
- **Match Power** is calculated:
  - Base: 1
  - +1 per tile beyond the minimum
  - Optional combo/upgrade bonuses
- **Blight destroyed = Match Power**
- Blight removal is applied to tiles in the group (prioritized arbitrarily or evenly)
- Any remaining blight on those tiles:
  - Is removed from the board
  - Returned to **Round Blight Pool**

### 6.4 Blight Interaction During Battle
- Any tile still carrying blight manifests an **Enemy Outpost**
- Destroying an outpost:
  - **permanently eradicates all blight it contains**
  - increments “blight cleared in battle” accounting

### 6.5 System Architecture Notes (Current Direction)
Blight responsibilities are being separated into focused systems:

- **BlightConverter (Battle-only):**
  - At battle start: converts tile.blight into enemy structures (e.g., Outposts)
  - Future: could map blight level → structure type, not just outposts

- **BattleRoot (battle accounting only):**
  - When an outpost dies: updates battle snapshot values (tile.blight=0, board.blight_cleared_battle += value, etc.)
  - In general: BattleRoot should coordinate systems and track results, not “own” blight logic.

---

## 7. Match-3 Mechanics

### 7.1 Matching Philosophy
- Minimum trigger: **3+ in a straight horizontal or vertical line**
- Matches are **group-based**:
  - A tile only counts as matched if it is part of a straight line of 3+ (horizontal or vertical) of one biome
  - Any two matched tiles of the same biome that touch orthogonally (up/down/left/right) merge into one group — so lines that share a tile (L, T, plus), sit stacked, or are offset by one all form a single group
  - A lone same-biome tile touching a group but not itself in a 3+ line does NOT join (Puzzle & Dragons-style; decided 2026-09-24, pinned by `MatchingRulesTest`)
  - No diagonal adjacency, ever

### 7.2 Match Resolution Flow (Wave-based)
Matches resolve in waves:
1. Detect all valid matches
2. Group connected matches
3. Resolve groups simultaneously
4. Apply blight destruction using Match Power
5. Remove tiles
6. Collapse and refill
7. Repeat until stable

Each wave increments a **combo count** for potential bonus scaling.

### 7.3 Player Interaction
- Fixed number of moves per round
- Default move: swap with adjacent tile
- Optional future move types (upgrades/leaders):
  - multi-swap before resolution
  - drag-to-match (Puzzle & Dragons style) for limited duration

---

## 8. RTS / Tactical Layer

### 8.1 Battle Timing
- Battle runs on a **real-time timer**
- Worlds/leaders can modify timer behavior (pause on abilities, longer/shorter rounds)

### 8.2 Terrain in Battle
- Gems convert into static terrain tiles for the battle duration
- Terrain influences:
  - movement speed multipliers
  - walkability (e.g., Water blocks ground by default)
  - gathering eligibility

World rules can override terrain behavior (Water→Ice passable, etc).

**Ground-unit terrain rules (decided 2026-09-24):**

| Terrain | Ground units |
|---|---|
| Field | full speed (×1.0) |
| Forest | three-quarter speed (×0.75) |
| Mountain | half speed (×0.5) — passable |
| Water | solid wall — blocked |

- **Flying units** ignore all of it: they fly over water and mountains and are never slowed by a tile.
- **The tile itself decides speed and blocking.** Decorations (below) never do.

### 8.3 Terrain visuals: neighbor-aware tiles & decorations (future, cosmetic only)
Direction from the developer (2026-09-24), not built yet:
- **Decorations** are a layer on top of a tile's base art: trees on forest, bits of rock / mushroom patches on fields, etc. Placed by an algorithm, **seeded per tile** (position + world seed) so they are stable and never re-roll. They have **no gameplay effect** (no speed, blocking or damage); at most they jostle when units walk past (Stardew-Valley feel).
- **Neighbor-aware:** decoration and edge art lean toward neighboring biomes — a field beside a forest gets a few trees along the shared edge, more toward a corner where two forest sides meet. Adjacent mountains could draw one continuous ridge instead of two separate rocky tiles (Godot's TileMap terrain autotiling is the likely tool; battle terrain is already a `TileMapLayer`).
- **Battle view only.** Matching-phase gems stay visually consistent and instantly recognizable (a core readability rule for match games). Gems may gain *state* overlays — e.g. veins of darkness whose density shows blight level, sparkle for modifiers — but a gem's base look must not vary randomly.
- Hooks that already exist: `PropsRoot` in the battle scene and the unused `TileModifier.spawn_battle_props`. Older notes (`Docs/history-and-vision-notes.md`) already favor deterministic seeded props and "props as suggestions, not clutter".
- **Explore later, not planned: ridge blocking.** Units could not pass *between* two adjacent mountain tiles (they'd route around the edges). This changes gameplay, unlike decorations, and needs edge-level blocking in pathfinding (today a whole tile is solid or open), plus care that it can't wall the castle in and that walls are readable on a phone. If tried, consider a slow "steep pass" before a hard wall.

---

## 9. Structures (TileStructures + Views)

### 9.1 Structure Model (Logic vs View)
- **TileStructure (logic)** lives on TileEntity (`tile.structure`)
- **StructureView (visual)** is spawned for battle from `TileStructure.create_view()`

This separation is intentional for:
- clean save/state tracking
- consistent spawning from board state
- simpler “build” actions during battle

### 9.2 Enemy Outposts
- Spawn from tiles that still have Blight at battle start
- Each outpost represents the blight value on that tile
- Outposts:
  - spawn enemies on an interval (with start delay variance)
  - can block gathering on their tile
  - can be attacked/destroyed

### 9.3 Outpost Levels & Scaling (Planned)
- Outpost “level” derived from blight value
- Higher blight → stronger/faster spawn patterns

### 9.4 Castle
- Central structure
- Does not participate in matching
- If Castle tile has Blight: triggers battle penalties (rules TBD)

**Future idea — castle in a match = next-battle buff (developer, 2026-09-25; not built, nothing decided beyond this):**
- Today the castle never matches (it breaks a line). The idea: it *can* be part of a match group, but is **never destroyed or removed** — it stays put while the rest of the group clears (same "kept tile" pattern as 4/5-matches).
- Reward: a **buff for the next battle round only**, scaled by the size of the group the castle was in. Position in the line doesn't matter (first, middle or last of a 3 all give the base buff).
- Buffs stay small, since they last one round and the castle is easy to include. Sizing example given: a base of ~2% for a group of 3, growing to ~20% for a block of 9 — note that is *steeper than linear* (3× the tiles, 10× the buff), so the scaling curve is a real design decision.
- **What gets buffed depends on the biome the castle sits on (developer answer, 2026-09-25):** Mountain/rock → defense (e.g. HP up); Forest → damage; Field → not settled — candidates are gather speed, all unit stats, or a free gatherer (a free gatherer per type is awkward to scale, so gather speed is the leading idea).
- **Scaling curve:** steeper-than-linear growth is acceptable *because* a big block (e.g. 9) is hard to make, especially without a drag-style move. The 2%→20% figures are placeholders; a much higher top end (e.g. 50%) is probably too much. To be tuned in playtesting, not decided.
- **Still open:** whether blight on the castle tile interacts; whether a matched castle affects the matching rules pinned in §7.1 (it would be the one tile that joins a group without being destroyed).

---

## 10. Units & Combat (Current + Direction)

### 10.1 Unit Types
**Player:**
- Combat units (e.g., Knight, Archer)
- Gathering units:
  - Farmer (Field → Food)
  - Lumberjack (Forest → Lumber)
  - Miner (Mountain → Stone / Gems)
  - Fisherman (Water/Ice → TBD)
Gatherers are vulnerable and require protection.

**Enemy:**
- Spawned from outposts
- Enemy type influenced by outpost biome + level (planned)

### 10.2 Current Implementation Model (v1)
- Units are Node2D with:
  - `UnitBrain` instances (PlayerBrain, EnemyBrain, GatherBrain)
  - basic targeting (nearest enemy in aggro radius; else structures/castle)
  - simple attack cooldown and melee/ranged logic
  - simple movement with “walkable” checks and terrain speed multipliers
  - basic separation to reduce clumping
- Projectiles are requested by units and spawned by BattleRoot.

### 10.3 Flying Units (Confirmed Requirement)
The design must support:
- Flying movement ignoring terrain costs and water blocking
- Targeting rules that distinguish **AIR** vs **GROUND**
- Units may only attack targets they can hit (anti-air constraints)

**Implementation direction:**
- Add unit domain (AIR/GROUND) and “hit mask” (can hit air/ground)
- Brains filter targets by `unit.can_hit(target)` before choosing nearest

### 10.4 Combat Upgrade Direction (Near-term)
Before adding full behavior trees:
- Add collision-aware movement (Area2D / CharacterBody2D if needed)
- Add attack “range sensing” using Area2D or distance gates
- Add consistent target selection + stopping logic (avoid orbiting/hugging)
- Keep brains simple FSM-like for now (behavior tree optional later)

### 10.5 Unit brains & movement — as built (2026-09-24)
Replaces the "v1" description in §10.2. Established after the first real playtest showed units freezing against lakes.

- **Movement:** A* over the tile grid (`BattlePathfinder`); water is solid. Straight line when clear, else cached waypoints, re-planned on a ~0.35s timer or when the goal changes tile. If a target is walled off, units path to the closest reachable point (archers can still shoot across a lake). No attack "slots" — units path to *reach* and stop.
- **Reach:** one shared rule on `Unit` (`melee_reach`, `ranged_reach`, `approach_reach`), body radius included for units, castle, outposts and chests. Movement stops walking and combat may swing at the same distance.
- **Spacing:** units collide with terrain only; soft separation keeps them from stacking.
- **Brains** (`CombatBrain` → `PlayerBrain` / `EnemyBrain`) only *choose a target*; think ~2.5×/sec (staggered per unit), immediately on target loss. Modes: IDLE, FIGHT_UNIT, PURSUE_OBJECTIVE.
- **Player fighter priority list:** (1) finish the enemy unit already being fought unless it leashes out (1.5× aggro) → (2) nearest enemy unit inside aggro radius → (3) keep hitting the current outpost while it stands → (4) next outpost = the one **closest to the castle** (defend-home-first, focused fire) → (5) hold position.
- **Enemy raider priority list:** (1) finish the player unit already being fought → (2) nearest player unit in aggro range → (3) march on the castle.
- Future hooks (not built): air/ground `can_hit` filtering (§10.3), castle-defense override, per-unit-kind priorities (e.g. archers prefer ranged threats).

---

### 10.6 Gatherer haul loop & raider targeting — DECIDED 2026-09-25, NOT BUILT YET
**Gatherer haul loop** (replaces today's "stand on a tile and collect 1/s forever"):
- A gatherer walks to a tile, spends **~3s gathering** (tunable, reference game was 2–5s), carries **1 item** (capacity 1 to start; carry capacity is a future upgrade), walks back to the **castle**, drops it off **instantly**, and repeats.
- Effect: income depends on the distance between resource tiles and the castle, so the map the player builds in the matching phase shapes the economy. Rough on-paper yield: ~5–6 items per gatherer per 30s battle (vs ~30 today).
- **Gather spots:** a bare tile has about **4 gather spots** (gatherers spread out around it; a full tile sends the gatherer to the next-nearest tile). Buildings on a tile change the spot count (a Mine starts with fewer and upgrades up to more). Numbers are arbitrary starting points.
- Later upgrade axes: carry capacity, gather time, move speed.

**Raider behaviour** — vision, not omniscience. The castle is always the goal; raiders get *distracted* by what they can see (vision = aggro radius, ~2.3 tiles today; may need a smaller value for raiders).
- **Walking rule** (what it moves toward): 1) the target it is already committed to, 2) otherwise the nearest **gatherer** it can see, 3) otherwise the castle. A raider already hitting the castle is committed to it and does not wander off after a gatherer that is merely nearby.
- **Swinging rule** (what it hits when something is already within reach, no walking needed): **gatherer > fighter > structure**, chosen per swing. So a gatherer dropping off next to a castle-attacking raider takes hits, and a fighter that attacks it gets hit back — but fighters are not aggro magnets.
- Gatherers running back to the castle naturally pull chasing raiders into the defenders.
- Possible later: player fighters defending gatherers under attack.

## 11. Resources & Economy

### 11.1 Core Resources
- Food
- Lumber
- Stone
- Gems / rare materials
- **Magic Dust** (flexible currency)

### 11.2 Magic Dust
Generated from:
- Matches
- Gathering
- Clearing outposts
- Special tiles/events

Used during runs to:
- modify match moves (drag, multi-swap)
- activate leader abilities
- trigger special battle actions

Carry-in rules:
- Dust exists in global bank
- Each world/run has a carry-in cap
- Cap can increase via upgrades/leaders

Mid/late game:
- Castle Refinery converts some unused Dust into **Refined Magic**
- Refined Magic is used for permanent unlocks outside runs

---

### 11.3 Economy core decisions & metal/refinery vision — 2026-09-25
**Economy core (payout built 2026-09-25; haul loop and raider targeting still to build):**
- **Match payout follows the reference game (Beetle Battle-style) — BUILT 2026-09-25:** resources per matched group = its **match power** (3 tiles → 1 item, 4 → 2, 5 → 3, i.e. 1 + tiles beyond 3). Starting small makes each extra tile a bigger relative deal. Today the game pays 1 item per destroyed tile (3× more), which is why matching out-earns nothing and gatherers looked huge on paper.
- **Dust stays generous:** 1 Dust per destroyed tile. Prices are *independent* of income — a unit may cost e.g. 1 water + 1 wood + ~10 dust; dust's value is set by prices, not 1:1 with items. Costs will be tuned from targets like "purchases per round", using measured income.
- Reference-game fact check (developer replayed it): 3-match = 1 resource, 4-match = 2; gatherers walk to a tile, gather ~2–5s, carry it to the castle, instant drop-off, repeat; enemy units target gatherers before the castle; the reference had 5 tile types (so matches were rarer than our 4).
- The haul loop and raider targeting: §10.6.

**Vision (not committed):**
- **Mine** — a passive producer that ticks **only its deposit's special resource** (e.g. iron ore; later gems), never basic stone, so it can't recreate a sit-and-tick economy. Tick is **chance-based** (e.g. ~1s tick at a low % chance) with **chance** and **amount** as upgrades (chance is capped at 100%, tick speed fixed to avoid three multiplying axes). Gatherers sent to it give an optional boost/extra output, not a requirement. **Mines do not persist between rounds.** Expected output = chance × amount ÷ seconds-per-tick; sample start ≈ 1 ore, 1s tick, 20% ≈ 0.2 ore/s (~4–6 per battle) for a high cost. Why build one: mined ore banks into the meta stash for future runs, and can be a tier you have not stockpiled.
- **Castle Refinery** (a castle upgrade, always present — no build cost or time): refines a stockpiled ore into **bars** during every battle (e.g. 2 ore → 1 bar per 5s). Ore is carried into a run (carry-in cap, like Dust) and can come from the meta stash. Set to one recipe; **if it runs out it automatically advances to the next ore type the player owns (iron → silver → gold), and stops if none remain.** Throughput, not ore, is the limit: at the base rate ~6 bars per battle. Refinery upgrades (speed, efficiency, more ore types) are a castle upgrade path. Relates to the older "Castle Refinery: Dust → Refined Magic" (§11.2) — one refining mechanic family.
- **Metals & tiers:** iron (T1), silver (T2), gold (T3), then stranger fantasy metals. Ores are stockpiled but cannot be spent; **bars buy permanent meta upgrades/unlocks** (e.g. "Knight damage level 2 = 5 iron bars"), possibly mixed across tiers so older metals keep a use. Start with **iron only** and prove the whole chain before adding tiers. Deposit definitions are data files, so new ores are mostly data.
- **Bars are not a per-spawn cost.** Spawning stays on base resources + dust (so round 1 of an advanced run is never locked out). Possible later: elite units costing a bar or two, carried in under a cap.
- **Castle deposit selector:** castle upgrades shift deposit *odds* (e.g. "silver deposits 2× likely"), never guarantees; deposits are made in the matching phase so this links the two phases. Higher progress should bring higher-tier ore sooner in a run.
- **Castle auto-builder:** a very expensive castle upgrade that builds a Mine on a deposit at battle start (nothing is built if no deposit exists; maybe limited to one). It removes a decision, so keep it costly.
- **UI:** the battle HUD shows only battle resources (base resources + dust). Ore/bars appear only in a **castle panel** (tap the castle; it **pauses the battle and the refinery clock**) showing recipe, rate and time-until-empty, plus a small glanceable badge on the castle. The recipe is chosen on the resolution screen or meta menu, changeable mid-battle.
- Open: carry-out rules for bars on a lost run; whether the Refinery and the Dust refinery are one building; the Mine's real build time (not yet measured).

## 12. Progression & Meta Systems

### 12.1 Rogue-like World Structure
- A “world” is one run composed of multiple rounds
- Enemy pressure persists across rounds until purged
- Failure: Castle destroyed → run ends + post-mortem summary

### 12.2 Difficulty Escalation
Driven by **pressure concentration**:
- higher blight budgets per round
- higher average outpost levels
- distribution rule changes (clustering, castle bias, biome bias)
- enemy behaviors tied to blight (spread/resistance/etc)

---

## 13. UI / UX Principles
- Strong distinction between phases
- Clear communication of:
  - remaining moves
  - resources earned
  - blight pressure (tile + pool)
  - outpost threat levels
  - battle timer
- Battle attention should focus on:
  - enemy positioning + threats
  - spawn choices (combat vs gathering)
  - timing abilities (future)

---

## 14. Current Architecture Notes (BattleRoot Extraction)

BattleRoot is being slimmed down into “systems” nodes:

- **StructureSpawner**
  - Spawns all StructureViews from board state (`spawn_from_board`)
  - Wires StructureView signals (fx, destroyed, spawn_requested, etc)
  - Places views under correct team roots
  - Future: unify Merchant/Chest/etc into same pipeline (as TileStructures)

- **BlightConverter**
  - Converts blight on tiles into enemy TileStructures before spawn
  - Future: supports multiple enemy structure types from blight

- **MerchantSystem**
  - Places merchant shop based on round rules and viable tiles
  - Future: selects inventory, prices, rarity, events

BattleRoot remains the coordinator + result accountant:
- starts/ends battles
- manages unit spawn calls
- tracks battle report values
- handles “global” FX/projectiles

---

## 15. Open Questions / Parking Lot

- Final grid size (5×5 vs 7×7)
- Exact blight placement algorithm + bias controls
- Chest tile behavior (matching + battle)
- Long-term balance: Dust flexibility vs resource specificity
- Terrain adjacency props generation (visuals + collision) as future polish

---

## 16. Next Steps (Suggested Order)

1. Finalize match detection + grouping + wave resolution
2. Implement blight destruction application during match resolution
3. Stabilize battle map generation from tile terrain
4. Upgrade unit movement/combat foundation:
   - add air/ground targeting support
   - improve range stopping + targeting reliability
   - add collision-aware movement (incrementally)
5. Add first flying unit + first anti-air unit to validate design constraints
6. Revisit behavior tree vs FSM once unit variety increases

---

## Change Log
- 12/17/25: Clarified match grouping and combo wave resolution
- 2026 (recent): Began extracting BattleRoot into StructureSpawner, BlightConverter, MerchantSystem; confirmed flying + anti-air requirement; moved toward tile-driven terrain rendering and collision-aware battle movement.
---

## 17. Vision & Pillars (added 2026-09-21)

**Inspiration:** *Puzzle & Dragons* (pattern-finding depth, leaders that bend matching rules) fused with *Beetle Battle* (the board you match on becomes the battlefield). The RTS side is meant to be as rich as the puzzle side: not just one castle, but camps, towers and other objectives to take or destroy.

**Why the blight never fully goes away (in-fiction):** the player isn't cleansing the world, they're *containing* an inevitable force — holding a line, not winning a war outright. Every push back provokes a counter-push elsewhere; clearing a region stabilizes it, but the source keeps producing pressure. This is why blight escalates even while the player is winning individual rounds, and it's what lets a "world" have a real ending (stabilized) while an endless/late-game mode still makes sense (the containment never truly finishes). Recovered from an earlier design conversation (`Docs/history-and-vision-notes.md`).

**Pillars**
1. **The battle is a consequence of the matching.** What you match decides your economy, the terrain, and where blight becomes enemy outposts. If battle ever feels like a separate game, something is wrong.
2. **Blight is a resource to manage, not just an enemy.** Where to clear it, and when, is a decision.
3. **Leaders change the rules.** Board shape, castle terrain, passive abilities and dust-powered skills vary by leader and world.
4. **Roguelike per world, with meta progression** between runs.

**Boards:** dynamic in *size* (square or rectangle, varying by world or leader), but always solid, with **no holes or odd shapes**. Reason: puzzle predictability (players learn patterns, as in P&D) and simpler battle pathing. Open: minimum tile size on phones.

## 18. Biomes & Economy Decisions (added 2026-09-21)

### 18.1 Biome roles
| Biome | Matching gives | Battle role | Special spot |
|---|---|---|---|
| Mountain | Stone, chance of gems | Slows movement; gatherers hack stone | Mine (auto-yield on tick, chance of gem) |
| Field | Food | Fast, open ground; gatherers farm | Farm / crops (upgradable crop type) — planned |
| Forest | Wood | Slows movement | Firefly grove: generates Magic Dust — planned |
| Water | Water (life support for all units) | Blocks ground units | Fishing / docks / naval — **later** |

- Each biome gets a "special spot" that yields on a timer with a chance of something better. Build this as one generic system (the existing `DepositModifier`), not bespoke code per biome.
- Mountain rim-only walking: parked as "cool later".

### 18.2 Unit cost tiers
- Gatherers: cheapest (water only at first).
- Base units: add food.
- Beefy units: add stone/wood.
- Magical units: add dust or rarer materials.
- Unit **upgrades** are permanent meta-progression, bought outside runs with gems (e.g. emeralds), dust and other carried-out resources.
- Guarantee a minimum of water tiles on the board so a random layout can't make building impossible.

### 18.3 Magic Dust
- Flexible "oh no" currency: any unit can be bought **entirely with dust at a steep markup** (or entirely with resources). No mixed recipes for now.
- Also the fuel for leader skills.
- Earned 1 per matched tile, from gatherers, outposts, special tiles.

### 18.4 Blight and biomes
- Blighted tiles give a **bonus** when matched (not a penalty). Player decides when clearing is worth it.
- Watch: bonus must not be so large that leaving blight is optimal. Leftover blight still becomes outposts.
- Number of blight levels on a tile also scales outpost strength and may change how the biome functions.

### 18.5 Castle and leaders
- Castle terrain is **random each run, never water** — and a leader can force it (e.g. a forest leader always starts on forest).
- Each leader: one passive (always on) + dust-powered active skills. Each should be explainable in one sentence.

### 18.6 Carry-in / carry-out
- Start of run: bring in resources up to a per-resource cap.
- End of run: bring out up to a cap per resource type. **Losing** = lower caps; **winning** = higher caps.
- Caps are upgradable in the meta game; castle level also affects storage.
- Watch: don't stack too many different caps (carry-in, carry-out, in-run storage). Treat them as one "storage" idea.

### 18.7 Open
- Chest behaviour, forest "tree elf" structures, exact numbers for dust markup, blight bonus, caps.

## 19. Castle, Units & Leaders (added 2026-09-23)

### 19.1 Design principle: verbs vs. numbers
A recurring check for any progression system (units, leaders, castle, meta upgrades): **a new unlock should mostly add a new verb** (something you couldn't do before — a new unit role, a new ability, a new rule) **rather than just a bigger number** (more damage, more range, more resources). Numbers aren't bad — they're the natural way to scale a verb once it exists — but a progression track that is *only* numbers, round after round, is what made past reference games (see §17) feel like it "pittered out" at higher levels. This isn't a hard rule per unlock; it's a ratio to watch across a whole progression track (see 19.3, 19.4).

### 19.2 Castle
Two separate axes, not one system:
- **Terrain (per-run, temporary):** the castle's terrain is random each run (§18.5, never water), and gives a small terrain-flavored effect for that run (e.g. a mountain castle takes less siege damage). A leader can force a specific terrain.
- **Tier (permanent, meta progression):** the castle itself ranks up over meta progression — e.g. Motte-and-Bailey → stone keep with towers — unlocked with meta currency, not RNG collection. Tiering up should unlock at least one real verb (e.g. **garrisoning units** in the castle for passive defense), not just raise base HP/damage.
- These two axes combine: terrain gives per-run character on top of whatever tier has been permanently reached.

### 19.3 Units — role slots + tiered rank-up
- Units are organized into **role slots** (e.g. "heavy melee," "ranged," "gatherer"), not an open-ended roster. A slot can be filled by different unit lines with different strengths (e.g. Knight = balanced heavy melee, Paladin = anti-magic heavy melee) — this is itself a good source of strategic choice without needing a large unit count.
- Within a slot, a unit line **ranks up** over meta progression (Knight → Paladin → higher tier), each tier ideally adding a small kit change (a verb), not just bigger numbers. Numeric upgrades (bought with gems/emeralds + carried-out resources, see §18.2) scale whatever tier you're currently at.
- **Explicitly rejected:** a large per-unit ability list (e.g. 20 abilities, pick 2 per run) and RNG/gacha-style "collect N shards to unlock" progression. Both add more complexity and/or predatory-feeling grind than this project wants. A per-run "loadout" system is a possible future addition (see Parking Lot) but not v1.

### 19.4 Leaders
- **One leader per run, chosen at the start, no swapping mid-run.**
- **Kit shape:** a bounded kit — starting 1 passive + 1 active — with meta progression able to unlock a **2nd passive and 2nd active slot** (the verb-unlock layer). Within a run, abilities you have **grow stronger via a rising cap** (Hades-style: start a little higher / can reach further, the number-scaling layer). This cleanly splits "new verb" (permanent, between runs) from "bigger number" (temporary, within a run).
- Passive abilities can be matching-facing or battle-facing (or both); same for actives.
- **The leader can be physically present on the battlefield:** summonable by spending resources, provides an aura (buff to nearby player units or debuff to nearby enemies), and dying while summoned has a real cost (damage or a debuff to the castle) — a genuine risk/reward spend each battle, not a freebie.
- A rare, later-game idea: special abilities unlockable only via a super-rare resource (not designed yet, parking lot).
- **Control model (resolves a real tension with the existing "indirect control" pillar, §8):** regular units stay fully autonomous (brain-driven, as already built) — the game does **not** become StarCraft-style click-and-command for the army. **The leader is the one exception**: while summoned, the player can target its abilities directly (click an ability, then click a unit/outpost/location). This gives real tactical agency in a small, buildable surface, without reversing the indirect-control pillar or requiring a selection/command system.
- **Parked for later, explicitly out of scope for now:** themed worlds (ice/fire/lava/cloud) that change board rules, terrain behavior, or unit availability per world. Not a rearchitecture risk — leaders can already override board layout, so this is additive whenever we get to it.

### 19.5 Open
- Exact numbers: cap growth rate per run, garrison capacity, aura radius/strength, leader-death penalty size.
- Whether the 2nd-passive/2nd-active unlock could ever be found in-run instead of purely permanent (parked; starting with purely-permanent for simplicity).
- Per-run ability loadout system (Model A from the leader/unit discussion) — a possible deeper addition once the base game is proven fun, not v1.

## 20. Matching-Phase Moves & Special Moves (added 2026-09-23)

### 20.1 Baseline moves (everyone, every leader)
- **Swap with a cardinal (N/S/E/W) neighbor** — costs 1 move.
- **Swap diagonally** — costs 2 moves. Available to all leaders by default; not a leader-gated unlock.
- Both consume from the same per-phase move pool (`moves_left`, already built).

### 20.2 Special moves ("prepped spells") — full model
Mental model: **Magic Dust's existing stated purpose** ("modify matching moves — drag, multi-swap, etc.") given a concrete mechanic. Framed like D&D spell prep: a leader *knows* certain moves and can *prepare* a limited number of them per phase.

- **Each leader innately knows exactly one special move** (their signature — e.g. Drag, Teleport, Row/Column Clear). Different leaders know different signature moves.
- **A special move is prepared as one of your existing per-phase moves, not an extra action.** It does not add to your move budget — it converts one of your moves into a "may be cast as special instead of a plain swap" move. E.g. 10 moves, 1 slot unlocked → up to 1 of those 10 can be a special cast; the other 9 are plain swaps either way.
- **UI model:** prepped specials sit as idle orbs next to the leader. Click one to "arm" it (glows/sparkles), then your next board action executes that special instead of a plain swap, consuming one move and the orb.
- **You choose when to cast** — any point during the phase, not tied to a specific move number.
- **If a prepped special is never cast, nothing is lost or refunded.** There's no separate "charge" sitting idle to waste — it was always going to be one of your N moves either way, cast as special or as a plain swap. No carry-over between phases; charges simply reset (D&D-slot-style: an unused prepared spell doesn't roll over, and that's normal, not a design flaw).
- **No recharge cost/mechanic in the battle phase** — deliberately rejected as unnecessary complexity (would require a new resource/build/tick subsystem for a feel that auto-recharge-next-phase already delivers). Specials simply reset to available at the start of the next matching phase.

### 20.3 Slots vs. move-count growth — the verb/number split, applied here
Two independently capped dials, matching the general verb/number pattern from §19.1/19.4:
- **Special-move slots** (how many of your moves per phase *can* be cast as specials): unlocked **permanently** via meta progression, per leader. This is the verb layer — a new slot is a new capability. Rough cap: **3 slots** (exact number TBD; kept deliberately small so loadout choice stays meaningful).
- **Total moves per phase**: can **rise during a run** as a leader levels up, capped. This is the in-run number-scaling layer (Hades-style), same pattern as ability-cap growth in §19.4.

### 20.4 Scrolls — teaching a leader a move they don't know innately
- The pool of possible special moves (Drag, Teleport, Row Clear, Column Clear, more later) is **shared and small/closed** — not an open-ended catalog. This is why an "ability pool + loadout" approach (Model A, generally parked for combat units in §19.3) is safe *here*: the option set won't balloon, and it's about board interaction, not combat balance.
- Unlocking a move for the shared pool at all, and then **teaching it to a specific leader**, are found via events ("devil in a tile" style, not tied to any one leader) — **spell scrolls**.
- **A scroll teaches one specific leader** the move, not the whole roster. Re-teaching the same move to a different leader you like using requires another scroll for that leader.
- Before a run, you choose which known moves fill that leader's open slots (pre-run loadout planning).

### 20.5 Enemy leaders — confirmed passive
- **No active "turn" for the enemy leader during matching.** Rejected in favor of legibility (matching should stay readable — the board shouldn't change for reasons outside the player's control between moves) and simplicity (no enemy-AI-swap logic to build/tune).
- Enemy leader presence is narrative + a difficulty dial (their world has denser/higher-level blight, tougher outposts), not an interactive opponent during the puzzle phase.

### 20.6 Match-4 / Match-5 "keep" tiles — status check (mostly already built)
- **Already implemented**, close to spec: a match of 4+ keeps the center tile of the longest run and applies a modifier to it (`Board.apply_match4_mods_at` / `ModifierRules.modifiers_for_match4`); a match of 5+ instead places a Chest there (`Board.apply_match5_chest_at`). Match-5 wins over match-4 if both apply.
- **Gap:** match-4's modifier is currently mountain-only (adds an iron deposit); forest/field/water do nothing. Match-5's chest is generic, not biome-flavored.
- **Design intent (not yet built):** both should have a per-biome flavor — e.g. a mountain match-5 chest gives rare gems, a forest one gives extra dust, water's is still undecided (ties into water's open "what does it give" question from §18).

### 20.7 Open
- Exact special-move catalog beyond the three named examples (Drag, Teleport, Row/Column Clear).
- Exact slot cap (rough guess: 3) and in-run move-count growth curve.
- Per-biome match-4/match-5 flavors (needs the biome-role conversation to mature further, esp. water).
- Blight "puke" on match (chance to spread to neighbors when a blighted tile is matched) and corrupted-chest interaction (a chest that's been infected has a chance to give something bad/worthless) — good tension ideas, not yet designed in full or built.

## 21. Loot, Meta-Upgrade Pattern & Special Board Events (added 2026-09-23)

### 21.1 Loot tables — nested and shared
- Loot from any source (mine tick, chest, farm harvest, etc.) is resolved via a **top-level table per source** ("what category: base resource / gem / rare item"), cascading into **shared sub-tables** for the specifics ("which gem: quartz/ruby/emerald/topaz/..."). 
- Key rule: **sub-tables are shared across every source that can roll into them**, not duplicated per structure. Rebalancing "how rare is a ruby" should be a one-place change. `Resources/GemLootTable.gd` exists today as a flat, non-tiered stub — this is the direction to grow it in, not a new system.
- A chest rolls into the same category/sub-table system as a mine or farm, just with different top-level odds (e.g. a bigger chance of the better categories).

### 21.2 House pattern: "Unlock, then Invest" (formalizes §19.1's verbs-vs-numbers principle)
This pattern has now emerged independently across four systems, worth naming explicitly and reusing as the default template for any future progression system:

| System | Unlock (verb layer) | Invest (number layer) |
|---|---|---|
| Castle | Random terrain / permanent tier | — |
| Leaders | Permanent kit-slot unlocks | In-run rising cap on abilities |
| Special moves | Permanent slot unlocks | In-run moves-per-phase growth |
| Meta upgrade tree | **A tiered key item unlocks a node** | **A secondary resource (gems) levels that node up** |

**The meta upgrade tree, worked out in full:**
- Each node requires **one tiered key item** to unlock (e.g. a Golden Axe for a tier-1 lumberjack node, Platinum for tier 2, Iridium for tier 3). **One item per node, every time** — not a one-time global key, and explicitly not a stacked-duplicate gate (rejected: "collect 20 shards for one node" feels grindy because bad luck compounds on a single gate; needing one item per node spreads the cost across many gates instead, so the same rarity feels fine).
- Once unlocked, a node can be **leveled further** by investing a secondary resource (e.g. rubies) — this is where numeric scaling happens, cleanly separated from the unlock itself.
- **Upward conversion** keeps lower-tier materials useful late-game: e.g. 10 Platinum Axes → 1 Iridium Axe.
- **Fallback conversion**: any key item with no remaining use (tree fully unlocked) converts to a currency (dust or similar) rather than sitting dead.
- Likely generalizes per gatherer archetype: axe for lumberjack, pickaxe for miner, etc. — a consistent, learnable pattern.

### 21.3 Environmental formations (curated, not a general pattern system)
- **Explicitly rejected:** a P&D-style general shape-bonus system (any L/T/cross/square triggers a bonus). Known failure mode: too many arbitrary shape→bonus mappings turns into a memorization/wiki problem, which directly conflicts with the "matching should stay legible" goal (§19.1/pillars).
- **Instead:** a small, hand-curated set of formations where the effect is **diegetic** — obvious from what the shape visually is, no memorization required. Confirmed examples:
  - **Moat**: a ring of water tiles around the castle → the castle gains a moat for the rest of the run.
  - **Wall**: a line of mountain tiles → treated as a hard, impassable barrier in battle (instead of individually walkable-but-slow mountains).
- This becomes a **third axis for the castle**, alongside random terrain (per-run) and permanent tier (meta progression) — something earned mid-run through skilled play.
- Exact list of formations beyond these two: open, to design later, but the *rule* (diegetic only, small curated set) is locked in.

### 21.4 "Devil in the Tile" — special board events
A category, not a single mechanic — multiple flavors are good (keeps events feeling fresh), all built on the **same underlying plumbing**: a `TileModifier` with a counter that moves via Match Power, the same architecture already built for blight. This keeps the category cheap to extend once the first example exists.

**Confirmed example — the Dragon Egg (matching → battle handoff):**
- A special tile/modifier spawns with some chance during matching; resists normal removal and instead accumulates progress each time it's matched (reuses blight's "match power moves a counter" pattern, inverted in meaning).
- Must be visually distinct so a player who doesn't want to trigger it can reliably avoid it — this optionality is the point, not an accident.
- Once its threshold is crossed, it "hatches" at the start of the battle phase into an encounter (e.g. a dragon) — fight it for a reward, or don't build toward it if not ready.

**Confirmed example — the Altar (battle-only):**
- A structure that can spawn in battle; spending a cost (e.g. "sacrifice" units/resources) at it summons something to fight for a reward, dragon-egg-style payoff but battle-native.
- **Decision: this is built as a leader-cast ability once the leader summon/target system (§19.4) exists — not a new rally-point/unit-command system.** A general "click any tile to redirect regular units there" mechanic was considered and explicitly rejected: it would reopen the indirect-control question we deliberately closed by making the leader the sole exception. Sequencing: build leader summon + targeted abilities for their own sake first; the altar becomes a near-free addition afterward ("Sacrifice" is just one more leader-targeted ability).
- Note: as of this writing, **no player-facing targeting/priority system exists in the code at all** — `PlayerBrain` auto-targets nearest enemy, then nearest enemy structure, with zero player input. Confirmed by reading the code (2026-09-23), not assumed.

**Backlog of further flavors (same underlying architecture, not designed in detail yet):**
- **Cursed Idol** — same counter-modifier shape as the egg, inverted valence: matching it spreads extra blight to neighbors instead of paying off. Gives "chase" and "dread" tiles from one system.
- **Hidden Vein** — a deposit invisible until revealed by a nearby match; light scouting tension.
- **Meteor** — a ticking-clock event instead of a counter event (already active, resolve it before N rounds pass, rather than build toward it via repeated matching).

**Build order:** an egg-style event first (cheapest — reuses existing architecture, zero control-model risk, validates the category). The altar and further flavors follow once the leader summon/targeting system exists.

### 21.5 Open
- Exact loot table percentages and category lists (base/gem/rare tiers, what "rare items" actually are).
- Full formation list beyond moat + wall.
- Exact dragon-egg numbers (spawn chance, hits-to-hatch, encounter difficulty/reward).
- Whether the altar's summoned thing is always an enemy-to-fight, or could ever summon an ally (leaning enemy-to-fight for consistency with the egg, not yet confirmed).

## 22. Core Battle Loop (added 2026-09-23)

### 22.1 What "just enough defense" means
Not an overall power-level target — a **resource-allocation** feeling. Every unit/ability spent on offense or gathering is something not spent on defense, so the tension each battle is judging the board and deciding how much can safely be risked. "Overpowered" moments come from reading it right and getting away with a bold gather; the baseline feeling should be "I allocated enough to hold, and used the rest to gain ground" (the Beetle Battle reference feeling, §17).

### 22.2 Balance as a practice, not a spec
Numbers are found through playtesting and iteration, not solved by up-front reasoning. The design job is to keep the knobs cheap to turn (most already are, via `@export`), not to guess correct values now. The one recurring question worth asking about any single number: which side of the scale is it on (see below), not whether it "feels right" in isolation.

**The scale**, both sides move round over round; balance is the *relationship* between them, not either number alone:
- **Up (income & power):** match rewards, building tick yields, unit stats/upgrades, chest/loot value.
- **Down (cost & pressure):** unit/ability costs, blight seed + growth rate, outpost strength, enemy stats, time pressure (moves per phase, battle duration).

### 22.3 No board reshuffle — deliberately rejected, and why it's specific to this game
A normal match-3 can reshuffle freely because board state is disposable. **This game's board state is not disposable — it's the thing the player is deliberately building into a battle map**, so an automatic reshuffle would erase potentially many moves of intentional work. Rejected for the same reason: guaranteeing a match in every falling tile (secretly biasing spawns undermines deliberate terrain-building and is exploitable — "watch the top and keep matching").

**Important correction (2026-09-23): there is no "deadlock" state to detect, by design.** Unlike a normal match-3, a swap here is never required to produce a match — every adjacent swap is legal on its own (see §1: "swaps gems to gain resources, reposition threats, and partially reduce Blight"). A player may spend moves purely to reposition terrain/blight into a specific pattern for the coming battle, with no match at all. So "the player is stuck, no legal move exists" cannot happen, and no deadlock-detection code is needed. Deliberately **not building** it — see the corresponding Roadmap decision log entry.

**One player-controlled, opt-in tool remains** (not automatic, always chosen):
- **End the matching phase early** — a normal, always-available action. Its actual purpose is forfeiting unused moves once the player is satisfied with their board (e.g. they've already built the map they want and/or banked enough resources) — not a "stuck" escape hatch, since being stuck isn't possible here.

**Cut (2026-09-23): full board regenerate ("mulligan").** Originally specced as covering two cases — "I'm stuck" and "I dislike this board." The first case doesn't exist (see correction above), which leaves only the second: a variance complaint ("bad opening roll feels bad") that hasn't been observed in actual play yet, and that sits in mild tension with the very reason automatic reshuffle was rejected (undermining deliberate board-building) — a mid-round mulligan lets a player discard their own already-placed intentional work, which is fine as an opt-in choice but means the feature's only remaining justification is speculative. Consistent with §22.2 ("balance/design found through playtesting, not solved by up-front reasoning"): not building this until real play shows it's actually needed. If it comes back, it comes back backed by an observed problem, not a leftover justification for a bug that no longer exists.

### 22.4 Danger visibility
No fog of war — the player already saw the blight during matching, so outposts appearing at battle start is expected, not hidden information.

### 22.5 Outpost formation from blight — hybrid: 1:1 below a threshold, merged clusters above it
- **Blight at or below a threshold** (placeholder: 5, to be tuned) on a tile → standard 1:1 outpost, as already built.
- **Adjacent tiles each above the threshold always merge** (deterministic — not a chance roll, matching the "matching stays legible/predictable" principle applied everywhere else) into a **single outpost** at a representative tile, carrying the **combined total blight** of the whole cluster.
- **Killing a merged outpost kills all of the blight it carries, at once.**
- Gives battle maps natural variety (a scatter of small outposts to mop up, plus occasional bigger landmark threats) and gives matching a genuine reason to defuse a heavy tile *before* it touches another heavy tile and locks into something bigger.
- Exact threshold and merged-tile placement rule (for odd cluster shapes) are implementation/tuning details, not blocking the core rule.

### 22.6 Castle death — permanent, no in-round recovery
If castle HP reaches zero, the run ends immediately. This is the central stake: the castle must be protected at all costs, and there's no clawing back within a round. Ideas floated but explicitly parked (not decided, not designed): builder-placed permanent friendly tiles from cleared blight, a leader resurrection ability.

### 22.7 Full clears — rare/optional, not the norm, with a large but rarity-linked reward
Most rounds: the player is fending off aggression and hoping to pick off some outposts, not clearing everything. A full clear should be optional and meaningfully rewarded (placeholder example: permanently removing 25% of the *world's* remaining blight pool), but the **reward size and the achievability of a full clear are linked** — watch this during playtesting rather than lock in a number now; too-achievable full clears with too-large a reward would deflate the world's pressure curve faster than intended.

### 22.8 Confirmed existing mechanic (verified against code, matches original intent)
Match Power (1 + extra tiles beyond the minimum) is spent as a budget across a matched group to **permanently kill** blight; any blight left over on destroyed tiles returns to a **pending pool**, which is re-manifested onto newly-falling tiles **within the same round's cascade** (not held until next round). This confirms: matching mostly *mitigates and relocates* blight; the outpost in battle is where it's actually eliminated for good. No change needed — already built as intended.

### 22.9 Open
- Exact blight-merge threshold and full-clear reward numbers (tuning, post-playtest).
- Merged-outpost tile-placement rule for non-trivial cluster shapes.
- Whether/how additional reset charges are granted (leader ability vs. meta-upgrade vs. both).

## 23. Unit Roster & Progression (added 2026-09-23)

### 23.1 Full target roster (combat archetypes)
Six role slots as the long-term target: **Knight** (heavy melee), **Archer** (ranged), **Pikeman** (anti-cavalry — hard counter, weak elsewhere), **Cavalry** (fast melee), **Flying** (air), **Mage** (magic/AoE). Plus existing gatherer roles (Farmer, Miner, ...).

**v1 scope: Knight, Archer, Pikeman only.** Cavalry, Mage, and Flying are deferred — Flying specifically needs the air/ground targeting work that isn't built yet (roadmap Phase 5).

### 23.2 Unit progression — two independent axes (reuses the castle/leader pattern)
Resolved: **not** a large per-unit ability tree (e.g. 20 unique skills). Instead, the same bounded-kit shape already used for leaders, applied to units:

- **Tier axis:** Knight → Paladin → ... — unlocked via the "Unlock, then Invest" key-item system (§21.2). Changes the unit's base identity/stats, a new answer to the same role slot.
- **Ability-kit axis:** a **small number of slots** (start 1-2, cap ~3-4) filled from a **small, shared** ability pool (cleave, crit chance, leap, etc. — not a unique tree per archetype). Slots unlock permanently (verb layer); each equipped ability levels up with a secondary resource (number layer) — e.g. cleave starts at 25% of normal damage to extra targets, levels toward 100%.
- Rationale: reuses the exact mental model already established for leaders (bounded kit, unlockable slots, leveled abilities) — one system the player learns once and applies to both leaders and units, cheaper to build than a bespoke tree per archetype.

### 23.3 No new resource system for unit abilities
Unit abilities (cleave, crit, leap, ...) are built as **variations on the existing attack-cooldown system** (`CombatComponent`/`attack_cooldown`), not a separate stamina/resource a unit manages. E.g. "this attack occasionally cleaves" rides the existing attack timer. Deliberately avoids adding a new per-unit resource-management layer.

### 23.4 Theming
Player units and enemy units are thematically paired opposites, reinforcing the whimsical/cutesy aesthetic (§2) and giving instant at-a-glance team readability:
- Player Knight → **teddy bear**; Player Cavalry → **unicorn**.
- Enemy Knight-type → **goblin**; Enemy Cavalry-type → **skeleton horse**.
- Pattern generalizes to the rest of the roster as it's built out.

### 23.5 Open
- Exact ability-kit slot cap (rough guess 3-4).
- Full shared ability pool beyond cleave/crit/leap.
- Cavalry/Mage/Flying theming and exact tier names beyond Knight→Paladin.

## 24. Pacing: Round, Run & Session (added 2026-09-23)

### 24.1 Three separate clocks
- **Round**: one matching phase + one battle + resolution. The atomic, tunable unit.
- **Run**: how many rounds until victory or castle death. An *emergent* number (round length × rounds played), not something to fix directly.
- **Session**: how long a player sits down in one sitting. Should be flexible — a single round, or many chained.

### 24.2 Starting hypothesis (to test once playable, not a locked rule)
Keep round length short (placeholder target: ~2-3 minutes total). A satisfying run of ~8-12 rounds then naturally lands around 15-30 minutes of wall-clock time — which resolves both stated fears at once (not too short to feel like progress, not a multi-hour commitment) without treating them as separate problems to solve.

### 24.3 Why the meta layer doesn't need to "fix" a failed run
Reference point: the Beetle Battle inspiration (§17) had **zero permanent progression** and still felt good to repeatedly fail at around round 5-6 — proof the core loop is fun to repeat on its own. Everything designed on top (carry-in/out caps §18.6, the meta upgrade tree §21.2, castle tiers §19.2, leader kit unlocks §19.4) is *additive* to an already-fun loop, not a repair for a broken one. **The one real requirement:** early meta unlocks need to feel impactful quickly, not after dozens of failed runs, or the safety net won't feel present when needed.

### 24.4 Session flexibility requires real persistence (elevates an existing gap)
Supporting both "quick round, close the phone" and "long chained session" requires a run to be **closeable and resumable without losing progress**. Today, nothing persists at all, not even the permanent meta stash (`MetaBank` is in-memory only). This was previously an open question ("does the game even have a save system?"); it's now a **direct requirement** of the pacing model, not a nice-to-have — elevated accordingly in the roadmap.

### 24.5 Open
- Exact round-length and run-length targets (tuning, post-playtest, per §24.2's hypothesis).
- World-to-world arc (how many worlds, what changes between them, finite vs. endless) — smaller, deferred.

## 25. Round-End Upgrade System — Reconsidered & Confirmed (added 2026-09-23)
- **Keep the existing pick-1-of-3 `RunUpgradeDef` system as-is, structurally.** Reconsidered against a Cuphead-charm-card comparison; confirmed it's already correctly scoped as the **in-run numbers layer** (verbs are always permanent/meta, per §19.1/21.2 — nothing should grant a verb mid-run, so this system never needed to learn how to).
- It already mixes categories beyond unit abilities (world/castle/tower/per-unit targets exist side by side) — matches the stated worry about not wanting it to become ability-only. No change needed there.
- Growth path: just **add new enum entries** as new numeric dials get built (leader ability cap, special-move-count growth), not a structural change.
- **One real refinement, not urgent:** support same-category-different-level offers in one draw (e.g. three crit-chance cards at different levels), for the "safe vs. greedy" texture Cuphead has. Small, content-level work, do it when building actual upgrade content, not now.
