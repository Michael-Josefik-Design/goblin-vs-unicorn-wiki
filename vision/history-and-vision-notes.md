# History & Vision Notes (from old ChatGPT chats)

> **What this file is:** a digest of `Docs/GPT_chats/*.md` (old ChatGPT conversations from earlier development), pulled out by an AI skim, not verified against the current code or design doc. Treat every line here as **"here's what I was apparently thinking back then,"** not as a current decision. Cross-check against `Assets/GameDesignDoc.md` and `ROADMAP.md` before acting on anything.
>
> Source files: `Docs/GPT_chats/Partial.md`, `Docs/GPT_chats/Terrain.md`. Digested 2026-09-22.

## Known-stale items (superseded already — kept only for record)
- Grid size "5×5" and tile size "128px" — current code and our own decision is **7×7 default, 192px, odd sizes 5–9 via BoardLayout.**
- Castle HP "50 base" — current code (`CastleTileStructure`) has `max_hp = 20`.
- Blight "moves with tile vs. lives on grid spot" contradiction — **resolved**: blight lives on the `TileEntity`, moves with it. (This was literally resolved mid-chat, see below.)
- "Blight starts at 10, +1/round" — matches current `Board.gd` defaults, still current.

## Genuinely useful — worth a real conversation
- **"Containing, not curing" blight framing.** Reframes the win condition: you're not eradicating evil, you're holding a line, so escalating pressure feels earned instead of contradictory, and it doesn't undercut an endless mode. This resolves a real tension your current design doc doesn't yet explain in-fiction. **Recommend adding to the design doc's vision section.**
- **Metals tier (silver/gold/platinum) with a "digging deeper wakes something up" hook** — i.e., mining could itself add blight pressure. This is a genuinely different idea from what we discussed (we talked mountains giving stone + gem chance, not mining-adds-blight). Worth a deliberate yes/no.
- **Fireflies as "insight," not power** — reveal tiles, improve match prediction, bend RNG — versus metals as raw power. A clean asymmetry we hadn't specifically named this way (we'd landed on fireflies → dust, which still fits, but "insight" utility fireflies is a distinct, extra idea).
- **"Avoid crafting trees; use conversion contexts instead"** (Lumber Mill: wood→lumber, Smelter: stone+fuel→metal, Sanctuary: fireflies→blessings). A concrete stated dislike/constraint worth keeping in mind as we design refinement buildings.
- **Meta upgrades should be "new verbs, not bigger buckets"** — i.e. avoid flat +% upgrades, prefer upgrades that change how a tile/unit behaves (e.g. "forest tiles can spawn protective fireflies instead of neutral ones"). A real design principle worth adopting explicitly.
- **Tile modifier taxonomy**: Environmental (neutral: rocky/flooded/fertile/dry) vs. Blight (corrupted roots, tainted veins, withered soil, polluted springs — reduces efficiency, "the land still works but it's fighting you") vs. Opportunity ("deal with the devil" tiles: rich vein = double metal + faster blight spread, etc.). This is a clean 3-category system worth considering wholesale for the modifier system, which today only has one deposit modifier.
- **Deterministic prop generation** via seeded RNG `(tile_x, tile_y, world_seed)` so decorative props are stable/repeatable rather than re-rolled — relevant whenever we get to battlefield art/props.
- **Aesthetic direction, stated repeatedly:** painterly/watercolor, soft edges, props as "suggestions not clutter," visual clarity over spectacle, explicitly *not* pixel-art noise or over-animated chaos. Useful to hold onto once we touch art.
- **Grid A\* over NavigationServer2D**, reasoning: tile-based movement costs and future modifiers are much easier to reason about on a grid than with Godot's nav mesh. Relevant when we get to unit pathing/movement rework.

## Commander (leader) abilities — running list from old chats (Nov 2024)
Never implemented. `Leaders/Abilities/LeaderAbilityDef.gd` exists as an empty stub (id, display_name, icon, description — no effect data, no defs, nothing wired to `Leader`). These are dust-powered *active* abilities in the vision (see §18.5 of the design doc: leader = one passive + dust-powered actives) — distinct from a leader's built-in passive.

**Matching-phase:**
- **Foresight** — see the next tiles coming, before they spawn.
- **Tile Transformation** — change one tile to a chosen type.
- **Drag Move** — reposition tiles by dragging rather than single swaps (already in the design doc as a "future move type").

**Battle-phase:**
- **Meteor Shower** — area damage to units and structures.
- **Asteroid Strike** — direct damage to one enemy tile/structure.
- **Spawn Illusionary Units** — decoy units with no HP.
- **Specialized Unit Generation** — spawn a specific unit with a unique skill.

Status: **ideas only, none built.** Worth turning into actual `LeaderAbilityDef` resources once we design the leader/dust-ability system for real.

## Open questions that were never resolved (still open)
- Should blight be allowed to start on the castle tile at all? (`allow_blight_on_castle_start` exists as a toggle in code; no firm answer.)
- Are props purely cosmetic, or do some affect movement/blocking? (Same tension as our own "cool later" note on mountain-rim walking.)
- Diagonal-aware decorative paths between fields — flagged as something that gets messy fast; no resolution.
- Tower / Shrine / Wonder structures were named as future ideas, never designed.

## Stated dislikes / constraints (good to keep enforcing)
- No crafting trees / deep recipe chains.
- Avoid scope creep; V1 = simple and readable, V2+ = depth through *rules*, not content volume.
- Meta upgrades shouldn't be flat number buffs.
- Visual noise and over-animation are explicitly unwanted.

## Not carried forward (superseded by our recent conversations, listed for completeness)
- Cost tiers, dust-as-shortfall vs. dust-as-full-payment, carry-in/out caps, castle terrain randomness — all of this was worked out fresh in our recent conversation and is already in the design doc. The old chats don't add anything new here beyond what we already decided.

## Second digest pass (2026-09-23, earlier chat content added to the top of Partial.md)
This pass covered older content the developer scrolled back and added. Unlike the commander-abilities find, nothing new turned up here — it's mostly the same early implementation planning already reflected in the current code (chain multipliers, cascade flow, castle-at-center, blight pool basics) or already-flagged-stale numbers (5×5 grid, castle HP 50). One thing worth noting: the **chain multipliers (1.0× / 1.25× / 1.5× / 2.0× for chain 1/2/3/4+) were a deliberate design choice**, confirmed in this chat, not an arbitrary value — recorded in the roadmap.

## How to use this file
When we start a new topic (units, structures, art, meta upgrades), it's worth a quick check here for "did past-me already have an opinion on this?" before we design from scratch. Nothing here is binding — it's a memory aid, not a spec.
