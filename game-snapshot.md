# Goblin vs Unicorn — game snapshot

_Generated from the game files on **2026-09-25** (game repo revision `f0ab3a6`). Numbers are exported straight from the game; rules text is hand-maintained. For design *intent* see `GameDesignDoc.md`; for status/plan see `ROADMAP.md`._

Status tags used below: **built** (in the game and tested), **planned**, **idea** (not committed to), **cut**.

## The story (developer's narrative)
A Shrek-style mash-up fairy-tale world where every story creature lives together, good and evil. **Good side:** Santa, leprechauns, the Tooth Fairy… **Evil side:** witches, mummies, Dracula… The **Blight is pure evil**: it powers the evil creatures and is trying to take over the good side's world. The player leads one of the good side's champions (the leaders) and defends a magical castle. Upgrading the castle can change the world around it (e.g. which ores appear). Real name and full plot are not written yet.

## What the game is
A round-based strategy game. A **Matching phase** (Puzzle & Dragons-style tile matching) earns resources and destroys *blight*; a 30-second **Battle phase** turns the same board into a battlefield where leftover blight becomes enemy outposts that spawn raiders and the player's units act autonomously (indirect control, no click-to-move); then a **Resolution** screen. Runs are several rounds; a Magic Dust stash persists between runs. Godot 4.5, GDScript, portrait mobile.

## Round flow (built)
1. **Matching:** 3 moves. Swap two adjacent tiles; every swap is legal whether or not it matches. Ends when moves run out or the player ends it early (confirm dialog).
2. **Battle:** 30 seconds. Board becomes terrain; blight left over becomes outposts; units fight and gather on their own.
3. **Resolution:** report + pick one run upgrade; next round has more blight.

Board 7x7, 192px tiles, castle in the center. Blight pressure starts at 5 and grows by 1 per round; the world starts with 10 blight.

## Matching rules (built, pinned by MatchingRulesTest)
- A tile counts only if it is in a straight horizontal or vertical line of **3 or more** of one biome. There is **no upper cap** on line length.
- Counted tiles of the same biome that touch **orthogonally** (up/down/left/right) merge into **one group**: plus, L, T, stacked and offset-by-one lines all merge.
- A lone tile touching a line but not in its own line of 3+ does **not** join.
- **No diagonal** adjacency, ever. Lines touching only at a corner stay separate groups (both still clear).
- The castle never matches; it breaks any line it sits in.
- **Match power** = 1 + (tiles beyond 3). It is a budget that permanently kills blight (1 per point) on tiles in the group. Blight left on destroyed tiles goes to a pending pool and reappears on new tiles that round.
- **Payout (built 2026-09-25):** each group pays its **match power** in items of its biome's resource (Field=food, Forest=wood, Mountain=stone, Water=water): 3 tiles -> 1 item, 4 -> 2, 5 -> 3, 6 -> 4. Each *destroyed* tile also pays 1 Magic Dust (a kept 4/5-match tile is not destroyed, so a 4-match pays 3 dust, a 5-match 4). Modifiers may add bonus yield. Before this change the game paid 1 item per tile.
- **Kept tiles:** the longest straight run in a group decides. A run of 4+ keeps one tile (may receive a modifier); a run of 5+ keeps a chest. The kept tile is `run[len/2]`, so in even-length runs it is the tile just past center (right/bottom). _Open design question: whether the kept tile should instead be the one the player moved._ Longer runs (6, 7) give no bigger reward than 5.
- **Cascades:** after clearing, tiles fall and refill; new matches resolve as another wave, repeating until stable. Each wave shows "Chain xN". Chain count gives **no** reward multiplier yet (planned).

## Terrain (built; speeds from code)
| Terrain | Resource | Ground speed | Ground units |
|---|---|---|---|
| Field | food | x1.0 | full speed |
| Forest | wood | x0.75 | slowed |
| Mountain | stone | x0.5 | slowed |
| Water | water | — | blocked (solid wall; units path around) |

The tile decides speed/blocking, never decorations. Flying units (planned) ignore terrain speed and water/mountain blocking. Movement uses A* on the tile grid with water as solid cells.

## Units (built; numbers from the unit definition files)
| Unit | HP | Damage | DPS | Speed (tiles/s) | Range | Cost |
|---|---|---|---|---|---|---|
| Archer | 20 | 4 | 6.7 | 2.60 | 2.34 tiles | 1 food, 1 stone, 1 wood, 4 dust |
| Farmer | 10 | 1 | 1.7 | 2.19 | melee | 1 food, 1 water, 2 dust |
| Knight | 20 | 4 | 6.7 | 2.60 | melee | 1 food, 1 water, 1 wood, 4 dust |
| Miner | 10 | 1 | 1.7 | 2.19 | melee | 1 food, 1 water, 2 dust |
| Raider | 20 | 3 | 5.0 | 2.60 | melee | enemy unit |

Attack cooldown 0.6s for all units. Aggro radius 450px (2.34 tiles). **Reach** (one shared rule for movement and combat) = attack range + own radius + target radius, centre to centre. Combat numbers are **placeholder** and unbalanced.

**Brains** (re-think ~2.5x/second and commit to a choice):
- Player fighters: 1) keep fighting the current enemy unit (unless it runs beyond 1.5x aggro), 2) nearest enemy unit in aggro, 3) keep hitting the current outpost, 4) go to the outpost closest to the castle, 5) hold.
- Enemy raiders: 1) finish current fight, 2) nearest player unit in aggro, 3) march on the castle.
- Gatherers: find the nearest tile of their terrain (Farmer: Field, Miner: Mountain), walk there, collect 1 per 1.0s.

## Structures
- **Castle** (built): 20 HP; losing it loses the run; never matches. Blight on the castle tile causes battle penalties (rules TBD).
- **Enemy outposts** (built): made from leftover blight; 20 HP; spawn 1 raider every 2.0s while alive.
- **Lumber Mill** (built): base cost 5 stone, 5 wood; +1 stone, 1 wood per extra level (assumed linear).
- **Mine** (built): base cost 5 stone, 5 wood; +1 stone, 1 wood per extra level (assumed linear).
- **Iron deposit / mine yield:** 2 iron ore + 1 stone every 1.5s once a mine is running; gather chance 50% (+1%/level).
- **Chest** (partly built): a straight 5+ match leaves one; can't be matched; battle behavior/loot unsettled.

## Economy (numbers from code; income figures are on-paper, not measured in play)
Costs:

| Item | Cost |
|---|---|
| Farmer | 1 food, 1 water, 2 dust |
| Miner | 1 food, 1 water, 2 dust |
| Knight | 1 food, 1 water, 1 wood, 4 dust |
| Archer | 1 food, 1 stone, 1 wood, 4 dust |

Income facts: 3 moves/round; matching pays 1 dust per destroyed tile and each group pays its match power in items; a gatherer collects 1 per 1.0s during a 30s battle. Dust has **no battle income in code** (design doc says gathering should give some).
On paper a Farmer (2 items + 2 dust) can earn ~30 items per battle at 100% uptime while a full round of matching pays roughly 10 items, so gatherers may out-earn matching; a Mine (10 items) yields ~2 items/s once running. These are **unmeasured assumptions** (gatherer uptime, match groups per move, mine build time) and the main balance question.

**Measured 2026-09-25 (headless simulation of the real rules, `_Debug/EconomySim.gd`):**
- Matching, per round of 3 moves on a freshly built board (the game rebuilds the board every round): a random player makes a match on 20.7% of moves and earns ~4.8 items + ~4.8 dust per round today; a skilled bot (best swap every move) earns ~31.59 items + ~31.59 dust. Under the decided match-power rule items drop to ~1.91 (random) / ~13.55 (skilled). About half of all groups come from cascades (only 4 tile types).
- Gatherers today: 1 Farmer collects ~29.3 items per 30s battle (nearest field tile averages 2.78 tiles from the castle); 4 Farmers ~94.7 total. With raiders and only 2 Knights defending, the castle fell before 30s in some runs and most Farmers died.
- Mine: builds in 1.98s, then ~1.2 iron ore/s + 0.6 stone/s. It does NOT persist: the board is rebuilt every round.

Run upgrades that exist: Sharpen Arrows (+1 damage for archers); Fast Gather (Gatherers are 10% faster); Sharpen Swords (+1 damage for knights); Terraformer (More moves when matching); Seething Evil (Increase the blight per round). None have a cost yet.

## Decided but NOT built yet (economy core; step 1 measuring and step 2 match payout are done)
Everything above describes TODAY's behavior. These three changes are decided and planned (`plans/04-economy-core.md`); the economy numbers above will change when they land.
1. ~~Match payout = match power~~ — **built 2026-09-25** (see Matching rules above).
2. **Gatherer haul loop:** walk to a tile, gather ~3s, carry 1 item, return to the castle, instant drop-off, repeat; ~4 gather spots per bare tile. (Today: gatherers stand and collect 1/s.) On paper this cuts a gatherer's take from ~30 to ~5-6 items per battle.
3. **Raider targeting:** walking rule = committed target, else nearest visible gatherer, else the castle (a raider already hitting the castle stays on it). Swinging rule (things already in reach) = gatherer > fighter > structure, chosen per swing. Vision = aggro radius.
Prices will be set from targets like "purchases per round", using measured income, not from today's guesses.

## Vision, not committed (see GameDesignDoc.md §11.3)
- **Mine redesign:** passive producer of only its deposit's special resource (iron ore, later gems); chance-based tick with upgradable chance and amount; does not persist between rounds; ore banks into the meta stash.
- **Castle Refinery:** a castle upgrade (no build cost) that refines stockpiled ore into bars during every battle (e.g. 2 ore -> 1 bar per 5s); one recipe at a time, **auto-advancing to the next owned ore type (iron -> silver -> gold) when one runs out, stopping if none remain**; throughput-limited (~6 bars per battle at base). Bars buy permanent meta upgrades and unlocks, never per-spawn costs.
- **Metal tiers:** iron (T1), silver (T2), gold (T3), then fantasy metals; start iron-only.
- **Castle deposit-odds selector** and a very costly **castle auto-builder** for Mines.
- **UI rule:** ore and bars never appear on the battle HUD; they live in a castle panel (tap the castle; it pauses the battle).

## Ideas and open questions (not built)
- **Castle in a match -> next-battle buff (idea):** castle joins a group without being destroyed; buff scales steeper-than-linear with group size, capped; buff type by castle biome (rock=defense/HP, forest=damage, field=gather speed/TBD).
- **Terrain decorations (planned):** cosmetic, seeded per tile, battle view only, neighbor-aware edges; gems keep a consistent look, may gain state overlays (blight veins).
- **Ridges between adjacent mountains (explore later):** visual ridge; possible passage-blocking is a gameplay change needing edge-level pathfinding.
- **Flying units, merchant Trade/Shop, meta upgrade tree, chain-count rewards (planned).**
- **Cut:** mulligan; deadlock detection (every adjacent swap is legal, so no stuck state exists).
- **Open:** kept-tile choice in even-length runs; what limits gatherers; combat model (armor, damage types); castle-tile blight penalty.
