# Plan 04 — Economy Core

**Status:** DRAFT for developer approval (2026-09-25). Nothing here is built. The decisions behind it are recorded in `GameDesignDoc.md` §10.6 and §11.3 and the ROADMAP decisions log.
**Goal:** make the in-round economy behave like the reference game the developer replayed — matching pays a little, gatherers make trips, gatherers are exposed — then measure it and price things from targets, *before* the meta upgrade tree is hung on top.
**Out of scope:** Mine redesign, Refinery, metal tiers, castle selector/auto-builder (vision only, belongs with the meta tree).

## Why
Diagnosis (B8): matching pays 1 item per tile (3× the reference) and gatherers collect in place for ~30 items per battle (vs ~5–6 with a haul loop). Costs cannot be tuned against numbers that are structurally wrong.

## Steps (each independently shippable, tested, and committed separately)

### Step 1 — Measure the current baseline (read-only, no game changes) — ✅ DONE 2026-09-25
Headless simulations replacing the Balance Ledger's guesses: (a) random play for a few hundred rounds → average match groups per move, group size, cascade frequency (4 tile types), dust and items earned; (b) 30s battles with 1/2/4 Farmers → real items collected; (c) mine build time and whether a mine survives a round. Output: a "measured" table in the wiki.
**Acceptance:** numbers reproducible with a fixed seed; wiki economy pages show measured values.
**Result (`_Debug/EconomySim.gd` → `Docs/Wiki/measured-economy.json`):** random play ≈ 4.8 items/round, skilled ≈ 31.6 (today's rule) → ≈ 1.9 / 13.6 under match-power; ~50% of groups come from cascades; 1 Farmer ≈ 29 items/battle (today); Mine builds in 2.0s, yields 1.2 iron + 0.6 stone per s, does not persist (board rebuilt each round); with 2 Knights the castle often falls before 30s. Re-run after Steps 2–4.

### Step 2 — Match payout = match power — ✅ DONE 2026-09-25
Resources per group = 1 + tiles beyond 3; dust unchanged (1/destroyed tile). Update `_compute_group_rewards`, tests (`MatchingRulesTest` rewards), wiki/snapshot. Rewards popup text stays correct.
**Acceptance:** a 3/4/5 group pays 1/2/3 items (dust = destroyed tiles: 3/3/4 since the kept tile isn't destroyed); all suites green. **Done:** `_compute_group_rewards` pays `group_power` of the group's biome once; `MatchingRulesTest` pins 3/4/5/6.

### Step 3 — Gatherer haul loop + gather spots
GatherBrain states: go to tile spot → gather (~3s timer) → carry 1 → go to castle → instant drop-off → repeat (an FSM). Tile keeps ~4 claimable spots; a full tile sends the gatherer to the next-nearest tile. Carry capacity and gather time are unit-def fields (upgradable later). Efficiency: no per-frame scans; reuse the brain think-timer and pathfinder.
**Acceptance:** new `GatherLoopTest`: a Farmer on a tile N tiles away delivers at the expected rate; two gatherers on a 4-spot tile don't stack; existing tests green.

### Step 4 — Raider walk / swing targeting
Implement the two rules in `EnemyBrain`/combat (design doc §10.6): committed > nearest visible gatherer > castle for walking; gatherer > fighter > structure per swing when in reach. Raider vision as its own tunable (default = aggro radius). Extend `BrainTest`.
**Acceptance:** tests for: raider ignores a gatherer beyond vision, chases one inside it, stays on the castle once hitting it, takes free swings at an in-reach gatherer, retaliates against an in-reach fighter.

### Step 5 — Re-measure and set prices from targets
Agree targets with the developer (e.g. "~1 unit per round early, N by round 5"), re-run Step 1 sims on the new rules, then set costs/dust (dust is generous; prices independent of income). Update Ledger/wiki.
**Acceptance:** developer approves numbers after a playtest.

## Risks
Gatherers become vulnerable and unprofitable if raiders overwhelm them → tune raider spawn/vision in Step 5. Haul loop with a 30s battle may feel slow → keep gather time/capacity tunable. Each step changes balance; the wiki/snapshot update in the same commit.

## Questions still open
Exact gather time (~3s) and spot count (~4) are starting points. Raider vision radius (2.3 tiles today) may be too large. Whether dust should also come from battle (design doc says gathering; code doesn't).
