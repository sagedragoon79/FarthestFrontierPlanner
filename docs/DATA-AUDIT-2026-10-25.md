# Farthest Frontier Planner — data audit

**Date:** October 25, 2026  
**Planner audited:** `index.html` @ `5136b27` (153 building entries)  
**Reference data:** `data/BuildingCostsDump.csv` (223 entries, dumped April 16, 2026)  
**Mechanics reference:** `Assembly-CSharp.decompiled.cs` 2026-10-09 (game v1.1.2a, MD5 `9d59d5d9…`, matches your installed DLL)

Nothing has been changed. This is the report you asked for before any edits.

## How entries were matched

The planner uses its own ids (`town_center`), the dump uses game identifiers (`TownCenter`). Of 153 planner entries: **142** matched a dump row, **8** are planner-only work-area markers with no dump row by design, and **3** matched nothing. Matching used the `desirability.tag` field, normalized ids and names, and a hand-checked alias table; every pair was verified by footprint and materials.

Unit conventions, both confirmed against the decompile:

- Desirability bonus: dump stores a fraction, planner stores percent (`×100`).
- Radii: dump stores world units, planner stores tiles (`÷5`). `BuildManager._cellSize = 5f` at line 147302 confirms the divisor for v1.1.2a.

## 1. What the decompile confirms is correct

These were checked against game code, not the dump, and need no change:

- **The desirability formula is exactly right.** `HappinessManager` line 382940:

  ```
  if (dropoff) { num5 = bonus/2; num6 = bonus - num5;
                 num7 = 1 - dist/radius; contribution = num6*num7 + num5; }
  else         { contribution = bonus; }
  ```

  That is the planner's `(bonus/2) + (bonus/2)*(1 - dist/radius)`. The range test is inclusive (`dist <= radius`).
- **Tag deduplication is right in shape**, with one nuance below: non-`allowMultiple` buildings sharing a tag keep only one contributor, and `allowMultiple` buildings sum.
- **The tile divisor of 5 is right** for this build.
- **Deduplication already compares absolute values.** `calcShelterDesirability` uses `Math.abs(contrib) > Math.abs(byTag[tag].best)`, matching the game's `Mathf.Abs(num8) > Mathf.Abs(buildingToMaxBonus[building2])`, so a strong penalty correctly displaces a weaker bonus sharing a tag. An earlier draft of this report listed this as a defect; that was wrong.
- **All eight work-area radii are correct**, and the descriptions even cite the game values: rat catcher 18 tiles (`workRadius=90`), hunter 20 (`huntingRadius=100`), forager 20 (`foragingRadius=100`), fishing 6 (`fishingRadius=30`), work camp 15 (`workRadius=75`), arborist 8 (`harvestRadius=40`).

## 2. Mechanics that are wrong (highest impact)

These change what the planner tells you, not just a displayed number.

### 2.1 There is a fifth shelter tier the planner doesn't know about

The planner hardcodes four tiers at `index.html:2041-2043` — 0-29 Shelter, 30-64 Homestead, 65-84 Large House, 85+ Manor.

The game has **five**. `BuildManager.houseUpgradeReqs` is a four-element list of upgrade requirements (`UIWidget` lines 272152-272155 read `[0]`-`[3]` into sliders `desirabilityTier2`-`desirabilityTier5`), and the dump confirms rows for `Shelter`, `Shelter_Tier2`, `Shelter_Tier3`, `Shelter_Tier4` and `Shelter_Tier5`.

So the top colour band is wrong and one tier is missing entirely. The thresholds themselves come from `houseUpgradeReqs[i].locBonus`, which is serialized data — **the exact percentages cannot be read from the DLL** and need a runtime dump to pin down.

### 2.2 Market coverage radius is 3 tiles too small

The planner uses `radius:13` for the market coverage circle and `upgradeRadius:15` for Market Square. The game uses the market's **desirability radius** for residence service — `MarketBuilding.GetResidencesInRange(..., strategicPlanningRadius, ...)`, at all three call sites (lines 137413, 350433, 350479).

That radius is 80 world units = **16 tiles** base, and 90 = **18 tiles** at Tier 2. The planner's own `desirability` block in the same entry already says 16 and 18, so the coverage circle contradicts its own data. Since "shelter outside market radius" is one of the always-on warnings, shelters in a 3-tile band are being flagged wrongly.

### 2.3 Flag Pole lets you stack bonuses that don't stack in game

The planner sets `allowMultiple:true` on `flag_pole`; the dump says `False` for `Deco_Flagpole`. With `allowMultiple` false the game keeps only one contributor per tag, so a row of flag poles gives one bonus, not several. As written, the planner rewards a layout the game ignores.

### 2.4 Roads and hedges measure distance from the nearest road cell

For a `SplineRoadBuilding` the game replaces centre-to-centre distance with the distance to the nearest cell in the road's `SplineRoadContainer`. Hedge walls and trellises carry desirability and are spline roads, so a long hedge contributes from its closest point, not its origin.

## 3. Missing buildings

18 buildings exist in the dump with a real build-menu category and have no planner entry:

| Identifier | Category | Grid | Labor | Materials |
|---|---|---|---|---|
| `CatKennel` | Resource Production | 3×3 | 30 | 40 ItemPlanks, 15 ItemStone |
| `CedarTree` | Decorations | 1×1 | 12 | 5 ItemGoldIngot |
| `Deco_HedgeGarden` | Decorations | 4×4 | 20 | 60 ItemBrick, 20 ItemPlanks, 800 ItemGoldIngot |
| `Deco_TopiaryGarden` | Decorations | 3×2 | 30 | 40 ItemBrick, 500 ItemGoldIngot |
| `Deco_Trellis` | Decorations | 1×1 | 15 | 10 ItemPlanks, 10 ItemGoldIngot |
| `DeepMine_Clay` | Resource Production | 3×3 | 30 | 50 ItemLogs, 50 ItemIron, 1000 ItemGoldIngot, 1 ItemHeavyTool |
| `DeepMine_Coal` | Resource Production | 3×3 | 30 | 50 ItemLogs, 50 ItemIron, 1000 ItemGoldIngot, 1 ItemHeavyTool |
| `DeepMine_Gold` | Resource Production | 3×3 | 30 | 50 ItemLogs, 50 ItemIron, 1000 ItemGoldIngot, 1 ItemHeavyTool |
| `DeepMine_Iron` | Resource Production | 3×3 | 30 | 50 ItemLogs, 50 ItemIron, 1000 ItemGoldIngot, 1 ItemHeavyTool |
| `DeepMine_Sand` | Resource Production | 3×3 | 30 | 50 ItemLogs, 50 ItemIron, 1000 ItemGoldIngot, 1 ItemHeavyTool |
| `DogKennel` | Resource Production | 3×3 | 30 | 40 ItemPlanks, 15 ItemStone |
| `Doghouse` | Resource Production | 1×1 | 10 | 10 ItemLogs, 20 ItemPlanks |
| `FieldstoneWall` | Walls And Roads | 1×1 | 2 | 1 ItemStone |
| `FishingDock` | Resource Production | 4×2 | 20 | 30 ItemPlanks |
| `FruitTreePeach` | Food Production | 2×2 | 20 | — |
| `FruitTreePear` | Food Production | 2×2 | 20 | — |
| `StorageBuilding` | Storage | 1×1 | 15 | 12 ItemPlanks |
| `WallFenceSecondary` | Walls And Roads | 1×1 | 2 | 1 ItemLogs |

The three pet buildings are the dogs-and-cats content. Note they were **already in your April dump** — the planner simply never ingested them, so no new dump is needed to add them.

A further 9 dump rows are correctly absent — category `NONE` means they aren't in the build menu: `Bush_Blueberry_Large`, `Bush_Blueberry_Med`, `Bush_Blueberry_Small`, `Bush_Sumac_Large`, `Bush_Sumac_Med`, `Bush_Sumac_Small`, `Dock`, `Explorer`, `FlattenTool`. `Dock` is the shelved boathouse content and `FlattenTool` is a terrain tool.

Two rows in that table deserve a caveat: `FruitTreePeach` and `FruitTreePear` are "missing" only because the planner collapses all fruit trees into one `fruit_tree` entry. That's a defensible simplification — all three are 2×2 at 20 labor — so treat the real count as 16 buildings plus an optional fruit-tree split.

### Planner entries with no dump row

| Planner id | Name | Size | Note |
|---|---|---|---|
| `fort` | Fort | 6×4 | Not a missing row after all: it is `isSubEntry:true`, the upgrade form of Barracks, so it maps to `Barracks_Tier2`. |
| `watch_tower` | Watch Tower | 1×1 | Also `isSubEntry:true` — the stone upgrade of Lookout Tower, so it maps to `TowerLookout_Tier2`, not a duplicate. |
| `small_juniper_bush` | Small Juniper Bush | 1×1 | Only one juniper row exists (`Deco_Bush_Juniper`), already taken by `juniper_bush`. |

The game also ships tier variants the planner doesn't model at all — `TowerLookout_Tier3`, `TownCenter_Tier2`-`Tier4`, `Theater_Tier4`, and roughly 30 other `_Tier2` rows beyond the 8 service buildings currently supporting the U toggle.

## 4. Construction costs

This is the largest category of error. Of 142 matched entries, **117 have at least one wrong value**, and costs account for most of it.

The planner stores costs as prose inside the `desc` string (`'… · Cost: 40 Labor, 50 Planks, 15 Stone'`), which is also why they drifted: there's no structured field to check against the dump.

Counts:

- **157** wrong quantities
- **33** cases listing a material the game doesn't use for that building
- **28** buildings showing no cost at all

The root cause shows in the vocabulary. Across all 153 entries the planner's descriptions mention Stone 68 times and Bricks 3 times, while the game data uses `ItemStone` 95 times and `ItemBrick` 61 times. The planner never mentions clay, flax, heavy tools, or glass — the game uses `ItemClay` 10 times, `ItemFlax` 3, `ItemHeavyTool` 12, `ItemGlass` 3. Later-game materials were largely replaced with stone.

Worked examples, planner versus game:

| Building | Planner says | Game data says |
|---|---|---|
| `school` | 40 labor, 40 planks, 20 stone | 80 labor, 60 planks, **30 clay**, 150 gold |
| `temple` | 120 labor, 300 planks, 300 stone, 1200 gold | 120 labor, **100** planks, **100** stone, 1200 gold |
| `windmill` | 50 labor, 60 planks, 25 stone, 290 gold | 50 labor, 60 planks, **250 flax, 1 heavy tool** |
| `theater` | 120 labor, 150 planks, 150 stone, 1250 gold | 120 labor, 150 planks, **150 clay, 30 logs**, 1250 gold |
| `storehouse` | 30 labor, 20 planks | **40** labor, **30 logs** |
| `bakery` | 40 labor, 30 planks, 15 stone | **30** labor, **50** planks, **25** stone |
| `quarry` | no cost shown | 30 labor, 50 logs, 50 iron, 1 heavy tool, 1000 gold |
| `well` | 20 labor, 5 stone | 20 labor, 5 stone, **3 logs** |

A recurring sub-pattern is a correct quantity against the wrong material: `chicken_coop` has logs and planks swapped (35/10 against 10/35), `cavalry_stable` has gold and stone swapped (50/800 against 800/50), `barn` lists 40 bricks where the game wants 40 logs, `charcoal_kiln` lists 50 planks where the game wants 50 stone, and `arborist` lists 50 stone where the game wants 50 clay.

### Buildings showing no cost at all

`basket_shop`, `blacksmith`, `book_binder`, `candle_shop`, `clay_pit`, `cobbler`, `crypt`, `fieldstone_wall`, `fletcher`, `fortified_wall`, `foundry`, `fruit_tree`, `furniture_ws`, `glassmaker`, `gold_mine`, `guild_hall`, `hedge_fence`, `hedge_fence_gate`, `iron_mine`, `library`, `palisade_gate`, `paper_mill`, `potter`, `quarry`, `stable`, `storage_depot`, `weaver`, `wide_palisade_gate`

### Full cost mismatch list

| Planner id | Dump row | Problem |
|---|---|---|
| `arborist` | `ArboristBuilding` | clay - vs csv 50 |
| `arborist` | `ArboristBuilding` | extra cost stone=50 not in csv |
| `armory` | `Armory` | gold 50.0 vs csv 300 |
| `armory` | `Armory` | stone 100.0 vs csv 50 |
| `bakery` | `Bakery` | labor 40.0 vs csv 30 |
| `bakery` | `Bakery` | planks 30.0 vs csv 50 |
| `bakery` | `Bakery` | stone 15.0 vs csv 25 |
| `barn` | `Barn` | extra cost bricks=40 not in csv |
| `barn` | `Barn` | logs - vs csv 40 |
| `birch_tree` | `BirchTree` | gold 12.0 vs csv 5 |
| `birch_tree` | `BirchTree` | labor - vs csv 12 |
| `cavalry_stable` | `CavalryStable` | gold 50.0 vs csv 800 |
| `cavalry_stable` | `CavalryStable` | stone 800.0 vs csv 50 |
| `charcoal_kiln` | `CharcoalKiln` | extra cost planks=50 not in csv |
| `charcoal_kiln` | `CharcoalKiln` | stone - vs csv 50 |
| `chicken_coop` | `ChickenCoop` | logs 35.0 vs csv 10 |
| `chicken_coop` | `ChickenCoop` | planks 10.0 vs csv 35 |
| `civic_monument` | `MonumentCivic` | extra cost planks=200 not in csv |
| `civic_monument` | `MonumentCivic` | iron - vs csv 500 |
| `civic_monument` | `MonumentCivic` | labor 600.0 vs csv 400 |
| `civic_monument` | `MonumentCivic` | stone 500.0 vs csv 200 |
| `coal_mine` | `CoalMine` | logs - vs csv 20 |
| `compost_yard` | `CompostYard` | logs - vs csv 15 |
| `corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench` | gold 1.0 vs csv 15 |
| `corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench` | planks 4.0 vs csv 1 |
| `corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench` | stone 15.0 vs csv 4 |
| `economic_monument` | `MonumentEconomic` | bricks - vs csv 625 |
| `economic_monument` | `MonumentEconomic` | extra cost stone=300 not in csv |
| `economic_monument` | `MonumentEconomic` | labor 600.0 vs csv 400 |
| `economic_monument` | `MonumentEconomic` | planks 625.0 vs csv 300 |
| `ext_corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench_Monument` | gold 1.0 vs csv 15 |
| `ext_corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench_Monument` | planks 4.0 vs csv 1 |
| `ext_corner_bench_plaza` | `Deco_SmallPlazaWithCornerBench_Monument` | stone 15.0 vs csv 4 |
| `ext_flower_urn` | `Deco_Urn_Monument` | bricks - vs csv 4 |
| `ext_flower_urn` | `Deco_Urn_Monument` | extra cost stone=4 not in csv |
| `ext_flower_urn` | `Deco_Urn_Monument` | iron - vs csv 5 |
| `ext_garden_trail` | `LongGarden_Monument` | gold 15.0 vs csv 25 |
| `ext_garden_trail` | `LongGarden_Monument` | labor 25.0 vs csv 20 |
| `ext_garden_trail` | `LongGarden_Monument` | stone - vs csv 10 |
| `ext_large_statue` | `LargeStatue_Monument` | iron 5.0 vs csv 65 |
| `ext_medium_garden` | `MediumGarden_Monument` | gold 15.0 vs csv 30 |
| `ext_medium_garden` | `MediumGarden_Monument` | labor 15.0 vs csv 10 |
| `ext_medium_garden` | `MediumGarden_Monument` | planks - vs csv 5 |
| `ext_medium_garden` | `MediumGarden_Monument` | stone - vs csv 5 |
| `ext_medium_plaza` | `MediumPlaza_Monument` | gold 10.0 vs csv 30 |
| `ext_medium_statue` | `MediumStatue_Monument` | gold 800.0 vs csv 1200 |
| `ext_medium_statue` | `MediumStatue_Monument` | iron - vs csv 35 |
| `ext_medium_statue` | `MediumStatue_Monument` | labor 50.0 vs csv 40 |
| `ext_medium_statue` | `MediumStatue_Monument` | stone 40.0 vs csv 35 |
| `ext_rose_garden` | `RoseGarden_Monument` | bricks - vs csv 30 |
| `ext_rose_garden` | `RoseGarden_Monument` | extra cost stone=30 not in csv |
| `ext_small_bench_plaza` | `Deco_SmallPlazaWithBench_Monument` | gold 1.0 vs csv 15 |
| `ext_small_bench_plaza` | `Deco_SmallPlazaWithBench_Monument` | planks 4.0 vs csv 1 |
| `ext_small_bench_plaza` | `Deco_SmallPlazaWithBench_Monument` | stone 15.0 vs csv 4 |
| `ext_small_garden` | `SmallGarden_Monument` | extra cost gold=10 not in csv |
| `ext_small_garden` | `SmallGarden_Monument` | labor 10.0 vs csv 5 |
| `ext_small_garden` | `SmallGarden_Monument` | planks - vs csv 5 |
| `ext_small_garden` | `SmallGarden_Monument` | stone - vs csv 3 |
| `ext_small_shrub` | `SmallShrub_Monument` | gold 10.0 vs csv 5 |
| `ext_small_shrub` | `SmallShrub_Monument` | stone 4.0 vs csv 2 |
| `ext_small_statue` | `SmallStatue_Monument` | gold 200.0 vs csv 500 |
| `ext_small_statue` | `SmallStatue_Monument` | iron - vs csv 15 |
| `ext_small_statue` | `SmallStatue_Monument` | labor 25.0 vs csv 30 |
| `fence` | `WallFence` | logs - vs csv 1 |
| `fence_gate` | `WallFenceGate` | logs - vs csv 4 |
| `festival_pole` | `Deco_FestivalPole` | extra cost planks=15 not in csv |
| `festival_pole` | `Deco_FestivalPole` | gold - vs csv 150 |
| `festival_pole` | `Deco_FestivalPole` | labor 25.0 vs csv 20 |
| `festival_pole` | `Deco_FestivalPole` | logs - vs csv 10 |
| `festival_pole` | `Deco_FestivalPole` | stone - vs csv 10 |
| `firewood_split` | `WoodCutterBuilding` | labor 20.0 vs csv 15 |
| `firewood_split` | `WoodCutterBuilding` | logs 10.0 vs csv 8 |
| `flag_pole` | `Deco_Flagpole` | flax - vs csv 25 |
| `flag_pole` | `Deco_Flagpole` | gold 5.0 vs csv 35 |
| `flag_pole` | `Deco_Flagpole` | labor 10.0 vs csv 8 |
| `flag_pole` | `Deco_Flagpole` | logs - vs csv 2 |
| `flag_pole` | `Deco_Flagpole` | stone - vs csv 5 |
| `flower_urn` | `Deco_Urn` | bricks - vs csv 4 |
| `flower_urn` | `Deco_Urn` | extra cost stone=4 not in csv |
| `flower_urn` | `Deco_Urn` | gold 15.0 vs csv 40 |
| `flower_urn` | `Deco_Urn` | iron - vs csv 5 |
| `fortified_gate` | `WallFortifiedGate` | extra cost planks=10 not in csv |
| `fortified_gate` | `WallFortifiedGate` | logs - vs csv 10 |
| `garden_trail` | `LongGarden` | gold 10.0 vs csv 25 |
| `garden_trail` | `LongGarden` | stone - vs csv 10 |
| `gazebo_plaza` | `Deco_GazeboPlaza` | bricks - vs csv 60 |
| `gazebo_plaza` | `Deco_GazeboPlaza` | extra cost stone=25 not in csv |
| `gazebo_plaza` | `Deco_GazeboPlaza` | planks 60.0 vs csv 25 |
| `goat_barn` | `GoatBarn` | logs - vs csv 20 |
| `grand_plaza` | `Deco_GrandPlaza` | bricks - vs csv 80 |
| `grand_plaza` | `Deco_GrandPlaza` | extra cost planks=100 not in csv |
| `grand_plaza` | `Deco_GrandPlaza` | extra cost stone=50 not in csv |
| `grand_plaza` | `Deco_GrandPlaza` | gold 2000.0 vs csv 600 |
| `grand_plaza` | `Deco_GrandPlaza` | labor 40.0 vs csv 20 |
| `graveyard` | `Graveyard` | stone - vs csv 1 |
| `healers_house` | `HealersHouse` | clay - vs csv 20 |
| `healers_house` | `HealersHouse` | extra cost stone=20 not in csv |
| `healers_house` | `HealersHouse` | gold 300.0 vs csv 100 |
| `juniper_bush` | `Deco_Bush_Juniper` | gold 12.0 vs csv 5 |
| `juniper_bush` | `Deco_Bush_Juniper` | labor - vs csv 5 |
| `juniper_bush` | `Deco_Bush_Juniper` | stone - vs csv 2 |
| `large_statue` | `LargeStatue` | iron 45.0 vs csv 65 |
| `lookout_tower` | `TowerLookout` | extra cost planks=25 not in csv |
| `lookout_tower` | `TowerLookout` | logs - vs csv 25 |
| `maple_tree` | `MapleTree` | gold 12.0 vs csv 5 |
| `maple_tree` | `MapleTree` | labor - vs csv 12 |
| `medium_garden` | `MediumGarden` | gold 10.0 vs csv 30 |
| `medium_garden` | `MediumGarden` | planks - vs csv 5 |
| `medium_garden` | `MediumGarden` | stone - vs csv 5 |
| `medium_statue` | `MediumStatue` | gold 500.0 vs csv 1200 |
| `medium_statue` | `MediumStatue` | iron - vs csv 35 |
| `medium_statue` | `MediumStatue` | stone 30.0 vs csv 35 |
| `military_monument` | `MonumentMilitary` | extra cost planks=1000 not in csv |
| `military_monument` | `MonumentMilitary` | iron - vs csv 200 |
| `military_monument` | `MonumentMilitary` | stone 200.0 vs csv 1000 |
| `oak_tree` | `OakTree` | gold 12.0 vs csv 5 |
| `oak_tree` | `OakTree` | labor - vs csv 12 |
| `palisade_wall` | `WallPalisade` | logs - vs csv 3 |
| `preservist` | `PreservistBuilding` | bricks - vs csv 100 |
| `preservist` | `PreservistBuilding` | extra cost gold=100 not in csv |
| `rat_catcher` | `RatCatcherBuilding` | extra cost stone=20 not in csv |
| `rat_catcher` | `RatCatcherBuilding` | logs - vs csv 15 |
| `rat_catcher` | `RatCatcherBuilding` | planks 15.0 vs csv 20 |
| `root_cellar` | `RootCellar` | extra cost planks=20 not in csv |
| `root_cellar` | `RootCellar` | logs - vs csv 10 |
| `root_cellar` | `RootCellar` | stone - vs csv 20 |
| `rose_garden` | `RoseGarden` | bricks - vs csv 30 |
| `rose_garden` | `RoseGarden` | extra cost stone=30 not in csv |
| `sand_pit` | `SandPitBuilding` | logs - vs csv 20 |
| `sand_pit` | `SandPitBuilding` | planks 20.0 vs csv 30 |
| `saw_pit` | `SawPit` | logs - vs csv 25 |
| `school` | `School` | clay - vs csv 30 |
| `school` | `School` | extra cost stone=20 not in csv |
| `school` | `School` | gold - vs csv 150 |
| `school` | `School` | labor 40.0 vs csv 80 |
| `school` | `School` | planks 40.0 vs csv 60 |
| `shelter` | `Shelter` | logs 15.0 vs csv 10 |
| `small_bench_plaza` | `Deco_SmallPlazaWithBench` | gold 1.0 vs csv 15 |
| `small_bench_plaza` | `Deco_SmallPlazaWithBench` | planks 4.0 vs csv 1 |
| `small_bench_plaza` | `Deco_SmallPlazaWithBench` | stone 15.0 vs csv 4 |
| `small_dirt_clearing` | `Deco_ClearedGround` | stone - vs csv 1 |
| `small_garden` | `SmallGarden` | extra cost gold=5 not in csv |
| `small_garden` | `SmallGarden` | planks - vs csv 5 |
| `small_garden` | `SmallGarden` | stone - vs csv 3 |
| `small_shrub` | `SmallShrub` | labor 3.0 vs csv 5 |
| `small_statue` | `SmallStatue` | gold 100.0 vs csv 500 |
| `small_statue` | `SmallStatue` | iron - vs csv 15 |
| `small_statue` | `SmallStatue` | labor 20.0 vs csv 30 |
| `small_statue` | `SmallStatue` | stone 15.0 vs csv 20 |
| `smokehouse` | `SmokeHouse` | stone - vs csv 8 |
| `spruce_tree` | `SpruceTree` | gold 12.0 vs csv 5 |
| `spruce_tree` | `SpruceTree` | labor - vs csv 12 |
| `stockyard` | `Stockyard` | extra cost stone=8 not in csv |
| `stockyard` | `Stockyard` | labor - vs csv 15 |
| `storehouse` | `Storehouse` | extra cost planks=20 not in csv |
| `storehouse` | `Storehouse` | labor 30.0 vs csv 40 |
| `storehouse` | `Storehouse` | logs - vs csv 30 |
| `tannery` | `Tannery` | extra cost planks=25 not in csv |
| `tannery` | `Tannery` | logs - vs csv 25 |
| `temple` | `Temple` | planks 300.0 vs csv 100 |
| `temple` | `Temple` | stone 300.0 vs csv 100 |
| `theater` | `Theater` | clay - vs csv 150 |
| `theater` | `Theater` | extra cost stone=150 not in csv |
| `theater` | `Theater` | logs - vs csv 30 |
| `town_center` | `TownCenter` | extra cost stone=16 not in csv |
| `town_center` | `TownCenter` | labor 10.0 vs csv 40 |
| `town_center` | `TownCenter` | logs - vs csv 16 |
| `trading_post` | `TradingPost` | logs - vs csv 15 |
| `vault` | `Treasury` | extra cost planks=50 not in csv |
| `vault` | `Treasury` | iron - vs csv 25 |
| `vault` | `Treasury` | stone 25.0 vs csv 50 |
| `wagon_shop` | `WagonShop` | logs - vs csv 15 |
| `wagon_shop` | `WagonShop` | stone 15.0 vs csv 5 |
| `well` | `Well` | logs - vs csv 3 |
| `wide_fortified_gate` | `WallFortifiedGateWide` | extra cost planks=15 not in csv |
| `wide_fortified_gate` | `WallFortifiedGateWide` | logs - vs csv 15 |
| `wildflowers` | `Deco_Wildflowers` | stone - vs csv 2 |
| `windmill` | `Windmill` | extra cost gold=290 not in csv |
| `windmill` | `Windmill` | flax - vs csv 250 |
| `windmill` | `Windmill` | heavy tools - vs csv 1 |
| `wooden_bridge` | `WoodenBridge` | extra cost logs=10 not in csv |
| `wooden_bridge` | `WoodenBridge` | gold - vs csv 5 |
| `wooden_bridge` | `WoodenBridge` | labor 10.0 vs csv 8 |
| `wooden_bridge` | `WoodenBridge` | planks - vs csv 10 |
| `work_camp` | `WorkCamp` | logs 10.0 vs csv 20 |
| `wrought_iron_fence` | `WallIron` | bricks - vs csv 3 |
| `wrought_iron_fence` | `WallIron` | gold - vs csv 15 |
| `wrought_iron_fence_gate` | `WallIronGate` | bricks - vs csv 18 |
| `wrought_iron_fence_gate` | `WallIronGate` | extra cost planks=18 not in csv |
| `wrought_iron_fence_gate` | `WallIronGate` | iron - vs csv 10 |

## 5. Footprints

13 buildings have the wrong grid size. Rotations are treated as equivalent, and resizable buildings are exempt, so these are genuine differences:

| Planner id | Dump row | Planner | Game |
|---|---|---|---|
| `academy` | `Academy` | 5×5 | 5×4 |
| `book_binder` | `BookBinder` | 3×4 | 2×3 |
| `cavalry_stable` | `CavalryStable` | 5×3 | 4×5 |
| `chicken_coop` | `ChickenCoop` | 3×3 | 4×2 |
| `crypt` | `Crypt` | 4×4 | 3×3 |
| `ext_medium_plaza` | `MediumPlaza_Monument` | 3×3 | 2×2 |
| `ext_rose_garden` | `RoseGarden_Monument` | 2×2 | 2×3 |
| `gazebo_plaza` | `Deco_GazeboPlaza` | 3×3 | 4×4 |
| `guild_hall` | `GuildHall` | 5×6 | 5×4 |
| `quarry` | `Quarry` | 4×4 | 5×4 |
| `rose_garden` | `RoseGarden` | 2×2 | 2×3 |
| `stable` | `Stable` | 5×3 | 4×3 |
| `storage_depot` | `StorageDepot` | 4×4 | 2×3 |

`storage_depot` is the worst of these: 4×4 in the planner against 2×3 in the game, which is more than twice the area.

## 6. Desirability values

**3 buildings are missing desirability entirely**, including two significant penalties:

| Planner id | Dump row | Missing value |
|---|---|---|
| `ext_small_shrub` | `SmallShrub_Monument` | bonus=2 radius=5 |
| `paper_mill` | `PaperMill` | bonus=-30 radius=15 |
| `quarry` | `Quarry` | bonus=-20 radius=10 |

`paper_mill` at -30% over 15 tiles and `quarry` at -20% over 10 tiles are large penalties. Omitting them makes the planner's shelter percentages optimistic for any layout near either building.

**14 wrong bonuses.** These are not random — garden, park and plaza decorations are inflated across the board, and each inflated base entry has a matching inflated `ext_` (monument) twin:

| Planner id | Dump row | Planner | Game |
|---|---|---|---|
| `ext_garden_trail` | `LongGarden_Monument` | 6.0 | 4 |
| `ext_large_park` | `LargePark_Monument` | 12.0 | 8 |
| `ext_medium_garden` | `MediumGarden_Monument` | 8.0 | 5 |
| `ext_medium_plaza` | `MediumPlaza_Monument` | 6.0 | 4 |
| `ext_small_garden` | `SmallGarden_Monument` | 5.0 | 2 |
| `ext_small_park` | `SmallPark_Monument` | 9.0 | 6 |
| `garden_trail` | `LongGarden` | 6.0 | 4 |
| `graveyard` | `Graveyard` | -15.0 | 0 |
| `large_park` | `LargePark` | 12.0 | 8 |
| `medium_garden` | `MediumGarden` | 8.0 | 5 |
| `medium_plaza` | `MediumPlaza` | 6.0 | 4 |
| `small_garden` | `SmallGarden` | 5.0 | 2 |
| `small_park` | `SmallPark` | 9.0 | 6 |
| `theater` | `Theater` | 8.0 | 5 |

`theater` matters most: 8% over 80 tiles in the planner against 5% over 60 tiles in the game, so both the strength and the reach are overstated for a major building. Ignore the `graveyard` row here and in the radius table below — see section 7 for why the dump's zero is probably the incomplete value, not the planner's.

**Wrong radii (more than a tile out):**

| Planner id | Dump row | Planner | Game |
|---|---|---|---|
| `graveyard` | `Graveyard` | 2.0 | 0 |
| `theater` | `Theater` | 80.0 | 60 |

**8 sub-tile rounding differences** — the planner rounds to whole tiles where the game value isn't a multiple of 5 world units (for example 36 units = 7.2 tiles shown as 7, and 48 = 9.6 shown as 10). Low impact, but worth a deliberate rounding rule rather than per-entry guesses: `ext_garden_trail`, `ext_medium_garden`, `ext_ornamental_tree`, `ext_rose_garden`, `garden_trail`, `medium_garden`, `ornamental_tree`, `rose_garden`.

## 7. What I could not verify, and why

Being explicit about the limits of each source:

- **Shelter tier thresholds.** `houseUpgradeReqs[i].locBonus` is serialized on a prefab. The code proves there are five tiers but not where the boundaries sit.
- **Whether the April values still hold at v1.1.2a.** The dump predates the current build by about two months. Code defaults I spot-checked agree with it, but `[SerializeField]` means a prefab can override any of them, and prefab data isn't in the DLL. Everything in sections 4 to 6 is "planner disagrees with the April dump", which is only "planner disagrees with the game" to the extent the dump is still current.
- **The `graveyard` entry.** The planner has -15% over 2 tiles with `tag:'graveyard'`; the dump has 0 bonus, 0 radius, no tag. Not necessarily a planner error: `GetStrategicPlanningBldgTag` falls back to the Unity object tag when `strategicPlanningTagOverride` is empty, and the dump never captured object tags. A blank `Tag` column means "no override", not "no tag".
- **`lookout_tower` radius 16 and `apiary` radius 14.** Neither matches a dump radius, but neither is necessarily wrong. The lookout is a `GuardTower` whose display uses `combatRadiusDisplay`, and `Apiary.circleRadius` is computed as `honeyBonusGridSize / 2 * AgricultureManager.resourceCellSize` — a derived value with no `*Radius*` float field for the dump's reflection to find.
- **`FieldstoneWall` versus `WallFieldstone`.** The dump has both (labor 2 and 3, each 1 stone). The planner's `fieldstone_wall` is mapped to `WallFieldstone`; which one the build menu actually uses needs checking.

## 8. Suggested order of work

1. **Move costs out of `desc` into a structured field** and generate them from the CSV. This fixes the largest error class and stops it recurring. It also lets the planner show a running total of materials for a whole layout, which it can't do today.
2. **Fix the mechanics in section 2** — fifth shelter tier, market radius, flag pole `allowMultiple`, absolute-value dedup. These change planning decisions.
3. **Add the 16 missing buildings**, pet buildings and deep mines first.
4. **Correct footprints and desirability** from the tables above.
5. **Re-run `BuildingCostsDump` against v1.1.2a** to confirm the April values and capture the tier thresholds. Everything above can proceed first; this is the verification pass, and it's the only way to close section 7.

## 9. What was applied

All of the above was fixed on October 25, 2026 and verified in a browser. The planner now reports v1.1.

- **Costs are structured data.** Every building carries a `cost:{}` field generated from the CSV, in the game's own material names. The prose `· Cost:` text is gone from every description, and the tooltip reads the structured field. Linear and resizable buildings carry `costUnit` (`section` or `tile`) because every one of them is 1×1 in the dump, so their cost scales with size.
- **A running material bill** now sits in the status bar: labour and gold inline, the full breakdown on hover. This is only possible because costs became data.
- **Footprints, desirability, and tier-2 values** corrected from the CSV across 142 entries. Sub-tile radii now carry their exact values (7.2 rather than 7).
- **Market coverage** is 16 tiles, 18 upgraded.
- **Flag Pole** no longer stacks.
- **14 buildings added**, including the three pet buildings and all five deep mines. They reuse the closest existing icon, since adding real ones needs an icon dump.
- **Shelter tiers** moved into a single `SHELTER_TIERS` constant with the fifth tier present and its threshold explicitly `null`. The help text now says a fifth tier exists and that its requirement is unconfirmed, rather than silently topping out.
- **`BuildingCostsDump` extended to v1.4.0** to dump `BuildManager.houseUpgradeReqs` to `ShelterUpgradeReqs.csv`. One game run now captures every tier's `locBonus` and closes the only open question above. It compiles clean.

Not changed, deliberately: `graveyard` desirability (section 7), the fruit-tree split, and `WallFenceSecondary` / `FieldstoneWall`, which are internal prefab variants rather than build-menu entries — `WallFenceSecondary` is identical to `WallFence` in every column, and `FieldstoneWall` lacks the Town Center T2 prerequisite that the real `WallFieldstone` carries.

The shelter capacity readout (`4/5/6/8`) still covers four tiers. Extending it needs the fifth tier's capacity, which comes from the same dump run.

---

Generated from `audit.py`. Totals: 117 of 142 matched entries had at least one mismatch; 18 buildings were missing; 3 planner entries matched no dump row.
